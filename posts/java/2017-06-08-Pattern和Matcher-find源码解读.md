---
title: "Pattern和Matcher.find源码解读"
date: "2017-06-08T09:49:10.000Z"
tags:
  - "java"
source_url: "https://gpdream.github.io/2017/06/08/java/Pattern%E5%92%8CMatcher-find%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/"
recovery_source: "html"
---
# Pattern和Matcher.find源码解读

Pattern和Matcher.find做的动作，以及Matcher.find可能带来的指数级性能下降原因分析.

## Pattern compile做的动作

Matcher对象需要Pattern，Pattern对象是由Pattern.compile创建出来。先看Pattern.compile。Pattern.compile(String regex)-> new Pattern(String regex) -> Pattern.compile(),核心代码在compile里，代码较多，上注释。\

```plain
/**
     * Copies regular expression to an int array and invokes the parsing
     * of the expression which will create the object tree.
     */
    private void compile()
```

compile方法会生成出一个对象树，是根据正则表达式生成出来的若干个的Node\

```plain
String regexStr = "((\\d|\\w|_)*wht_pboc\\.\\w+(_\\w+)*)";
//(?:\\d|\\w|_)(funcCode\\.\w+(_\w+)*)
Pattern pattern = Pattern.compile(regexStr);
Matcher matcher = pattern.matcher(content);
```

以上面这段代码为例，生成出来的树就可能是GroupHead Node->Loop Node->MatchLast Node。\
事实上生出出来的树要比这个复杂的多，大概是Start Node->GroupHead Node->ProLog Node->Loop Node->Group Head Node->Branch 巴拉巴拉……。\
如果正则有分支的话，比如正则为`((\\d|\\w|_)*wht_pboc\\.\\w+(_\\w+)*)|((\\d|\\w|_)*jiayu\\.\\w+(_\\w+)*)`这样，那Pattern就会有一个atom数组，这其实是一个Node数组，atom数组长度对应分支情况。

此时atom的数组长度就为2.事实上在第一个GroupHead Node里，因为有\d,\w,_这三种情况，GroupHead Node的atom长度就为3了。字符串match失败要所有的atom都不匹配才能满足。而当前节点match成功，则会调用node.next.match继续匹配，直到树走到尽头为止。

## matcher.find方法是如何找到匹配字符串的

还是以这个字符串和文本为例，假设运行代码如下\

```plain
//demo代码
    String regexStr = "((\\d|\\w|_)*wht_pboc\\.\\w+(_\\w+)*)|((\\d|\\w|_)*jiayu\\.\\w+(_\\w+)*)";
    String context = "import java.lang.*;\n" +
                "\n" +
                "pboc_crdt_otln_dlq_cnt\n" +
                "def xbeta = (-1.90408137652954)+wht_pboc.cc_y5dlq_mon12_pct*(0.73909217042419)+wht_pboc" +
                "+aaaa.jiayu*(0.10435815782414)+jiayu.cc_amt_use_pct*(0.92517485373273)+aaa" +
                ".aaa*(-0.0313185144899745)+wht_pssssboc.dddd*(-0.00111017949189794);\n" +
                "\n" +
                "def tgtscr=Math.exp(xbeta)/(1+Math.exp(xbeta))"
    Pattern pattern = Pattern.compile(regexStr);
    Matcher matcher = pattern.matcher(content);

    while(matcher.find()) {
        System.out.println(matcher.group())
    }
    }
```

matcher.find()方法会调用matcher.search(int from)方法.from的值是matcher的全局变量nextSearchIndex，该值默认为0，find到之后会把值改变为find成功后的位置。\

```plain
boolean search(int from) {
        this.hitEnd = false;
        this.requireEnd = false;
        from        = from < 0 ? 0 : from;
        this.first  = from;
        this.oldLast = oldLast < 0 ? from : oldLast;
        for (int i = 0; i < groups.length; i++)
            groups[i] = -1;
        acceptMode = NOANCHOR;
        boolean result = parentPattern.root.match(this, from, text);
        if (!result)
            this.first = -1;
        this.oldLast = this.last;
        return result;
    }
```

可以看到，处理逻辑的方法是parentPattern.root.match(this, from, text);\
`parentPattern.root`是Pattern的初始节点。刚才Pattern.compile生出出来的树，是有两个分支，两个分支的树基本一样，都是Start -> GroupHead ->Loop ……。\
让我们先看看Start.match()方法.\

```plain
boolean match(Matcher matcher, int i, CharSequence seq) {
            if (i > matcher.to - minLength) {
                matcher.hitEnd = true;
                return false;
            }
            int guard = matcher.to - minLength;
            for (; i <= guard; i++) {
                if (next.match(matcher, i, seq)) {
                    matcher.first = i;
                    matcher.groups[0] = matcher.first;
                    matcher.groups[1] = matcher.last;
                    return true;
                }
            }
            matcher.hitEnd = true;
            return false;
        }
```

matcher 里的i就是matcher.search(int from)里的值，matcher.to是整个匹配的字符串长度，minLength是该node可能的最小长度，像刚刚的GroupHead的最小长度为1.因此该段代码就是相当于从from开始循环到结尾尝试找到匹配字符串，找到了就立即返回。

这里的next是一个Group Head。我们再去看GroupHead的代码。\
先看GroupHead的注释,大概是说存储能find到的group的头是从哪个位置开始的。\

```plain
/**
     * The GroupHead saves the location where the group begins in the locals
     * and restores them when the match is done.
     *
     * The matchRef is used when a reference to this group is accessed later
     * in the expression. The locals will have a negative value in them to
     * indicate that we do not want to unset the group if the reference
     * doesn't match.
     */
    static final class GroupHead extends Node
```

Group Head的matcher方法,直接就调用了next.match\

```plain
boolean match(Matcher matcher, int i, CharSequence seq) {
            int save = matcher.locals[localIndex];
            matcher.locals[localIndex] = i;
            boolean ret = next.match(matcher, i, seq);
            matcher.locals[localIndex] = save;
            return ret;
        }
```

在当前这个demo里，此时的next是一个Loop Node，再去看Loop Node的match.\

```plain
boolean match(Matcher matcher, int i, CharSequence seq) {
            // Avoid infinite loop in zero-length case.
            if (i > matcher.locals[beginIndex]) {
                int count = matcher.locals[countIndex];

                // This block is for before we reach the minimum
                // iterations required for the loop to match
                if (count < cmin) {
                    matcher.locals[countIndex] = count + 1;
                    boolean b = body.match(matcher, i, seq);
                    // If match failed we must backtrack, so
                    // the loop count should NOT be incremented
                    if (!b)
                        matcher.locals[countIndex] = count;
                    // Return success or failure since we are under
                    // minimum
                    return b;
                }
                // This block is for after we have the minimum
                // iterations required for the loop to match
                if (count < cmax) {
                    matcher.locals[countIndex] = count + 1;
                    boolean b = body.match(matcher, i, seq);
                    // If match failed we must backtrack, so
                    // the loop count should NOT be incremented
                    if (!b)
                        matcher.locals[countIndex] = count;
                    else
                        return true;
                }
            }
            return next.match(matcher, i, seq);
        }
```

正常情况下count小于cmax，因此代码会走到body.mach(macher,i,seq)方法中，这里的body是个SliceNode.对应的方法\

```plain
boolean match(Matcher matcher, int i, CharSequence seq) {
            int[] buf = buffer;
            int len = buf.length;
            for (int j=0; j<len; j++) {
                if ((i+j) >= matcher.to) {
                    matcher.hitEnd = true;
                    return false;
                }
                if (buf[j] != seq.charAt(i+j))
                    return false;
            }
            return next.match(matcher, i+len, seq);
        }
```

Slice类的matcher中有个buffer，这个buffer是根据正则表达式生成的，假设我们的正则表达式是`String regexStr = "((\\d|\\w|_)*wht_pboc\\.\\w+(_\\w+)*)`那Pattern生成这个buffer的值就为`wht_pboc`.代码会尝试循环查找文本i之后的buf.length和buffer是否匹配,如果匹配则认为GroupHead是符合规则的，返回true继续下一步match。

在我们这个demo中，如果返回false，也不能排除是不符合规则的，因为head是任意长度的\w或者_,需要继续往下一个位置循环，直到出现明确不匹配字符、匹配成功或者文本末尾才能确定。如果分支树很多，且不能立即确定是否符合，则会有指数级的循环增长。

假设文本是 0123456789wht_pboc.这样，那需要循环的次数就是 分支数*10*3*n,(3是因为当前正则里，GroupHead自己有3个分支，ßßn为后面字符串和wht_pboc.前面相同的长度).如此多的循环完成之后才能确定GroupHead的match是否为true.
