[TOC]

# DFA算法

DFA即Deterministic Finite Automaton，也就是确定有穷自动机，它是是通过event和当前的state得到下一个state，

即event + state = nextstate

应用：可实现敏感词检测，将敏感词集合构成一颗颗tree，每个节点为字符和是否结束标识，先根据所有的敏感词build一个tree，然后通过深度遍历判断当前词是否为敏感词。

```java
@Data
@Accessors(chain = true)
@EqualsAndHashCode
public class SensitiveWordTree extends HashMap<Character, SensitiveWordTree> {
    private boolean endFlag = false;

    public static SensitiveWordTree buildTree(Set<String> wordSet) {
        SensitiveWordTree root = new SensitiveWordTree();
        for (String word : wordSet) {
            SensitiveWordTree tree = root;
            for (int i = 0; i < word.length(); i++) {
                char c = word.charAt(i);
                SensitiveWordTree nowTree;
                if (tree.get(c) == null) {
                    nowTree = new SensitiveWordTree();
                    tree.put(c, nowTree);
                } else {
                    nowTree = tree.get(c);
                }
                tree = nowTree;
                if (i == word.length() - 1) {
                    tree.setEndFlag(true);
                }
            }
        }
        return root;
    }

    public static Set<String> checkTxtBySensitiveWord(String txt, SensitiveWordTree root) {
        Set<String> words = new HashSet<>();
        for (int i = 0; i < txt.length(); i++) {
            String word = checkIndexAfterTxtBySensitiveWord(txt, i, root);
            if (word != null) {
                words.add(word);
            }
        }
        return words;
    }

    public static String checkIndexAfterTxtBySensitiveWord(String txt, int beginIndex, SensitiveWordTree root) {
        SensitiveWordTree nowTree = root;
        StringBuilder word = new StringBuilder();
        for (int i = beginIndex; i < txt.length(); i++) {
            char c = txt.charAt(i);
            if (!nowTree.containsKey(c)) {
                break;
            }
            word.append(c);
            if (nowTree.get(c).endFlag) {
                return word.toString();
            }
            nowTree = nowTree.get(c);
        }
        return null;
    }
    
    @Deprecated
    public static void checkWordBySensitiveWord(String txt, SensitiveWordTree root) {
        SensitiveWordTree tree = root;
        StringBuilder word = new StringBuilder();
        int len = txt.length();
        for (int i = 0; i < len; i++) {
            char c = txt.charAt(i);
            if (!tree.containsKey(c)) {
                break;
            }
            word.append(c);
            if (tree.get(c).endFlag) {
                System.out.println("敏感词：" + word.toString());
            }
            tree = tree.get(c);
        }
    }

    public static void main(String[] args) {
        SensitiveWordTree root = SensitiveWordTree.buildTree(new HashSet<>(Arrays.asList("王八蛋", "王八羔子")));
        checkTxtBySensitiveWord("王八蛋AA王八羔子B", root);
    }
}
```



