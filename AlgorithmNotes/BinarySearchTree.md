1. remember how to inorder, preorder, postorder travese a binary search tree.
2. a balanced bst is that the depth of the two subtrees of every node never differs by more than 1.
3. use recursion to break question into sub-problems (like left sub-tree and right-sub-tree), and sometimes use memorization to accelarator the process.
4. when we need to sum a target from middle point consecutively to a current point, we can use prefix sum.
5. A tree is also a graph, in-degree + out-degree = 0.

# in-order traversal

```java
List<TreeNode> order = new ArrayList<>();
private void inorderTraverse(TreeNode node) {
        if (node == null) return;
        inorderTraverse(node.left);
        order.add(node);
        inorderTraverse(node.right);
    }

private void inorderTraverse2(TreeNode root) {
        Stack<TreeNode> stack = new Stack<>();
        TreeNode cur = root;
        while (!stack.isEmpty() || cur != null) {
            while (cur != null) {
                stack.push(cur);
                cur = cur.left;
            }

            cur = stack.pop();
            order.add(cur);
            cur = cur.right;
        }
    }
```

# reversed in-order traversal

```java
List<TreeNode> order = new ArrayList<>();
private void reverseInorderTraversal(TreeNode node) {
        if (node == null) return;
        reverseInorderTraversal(node.right);
        order.add(node);
        reverseInorderTraversal(node.left);
    }

public List<Integer> reverseInorderTraversal(TreeNode root) {
    List<Integer> order = new ArrayList<>();
    if(root == null) return list;
  
    Stack<TreeNode> stack = new Stack<>();
    stack.push(root);
    while(!stack.isEmpty()) {
        TreeNode curr = stack.pop();
        list.add(0, curr.val);
  
        if(curr.left!=null) {
          stack.push(curr.left);
        }
  
        if(curr.right!=null) {
           stack.push(curr.right); 
        }
    }
    return list;
}
```

# pre-order traversal

```java
public List<Integer> preorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        if (root == null) return res;
        Stack<TreeNode> stack = new Stack<>();
        stack.push(root);
  
        while (!stack.isEmpty()) {
            TreeNode curr = stack.pop();
            res.add(curr.val);
      
            if (curr.right != null)
                stack.push(curr.right);
      
            if (curr.left != null) {
                stack.push(curr.left);
            }
        }
  
        return res;
    }
```

# post-order traversal

```java
public List<Integer> postorderTraversal(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        if (root == null) return res;
        Stack<TreeNode> stack = new Stack<>();
        stack.push(root);
  
        while (!stack.isEmpty()) {
            TreeNode curr = stack.pop();
            res.add(0, curr.val);
      
            if (curr.left != null)
                stack.push(curr.left);
      
            if (curr.right != null)
                stack.push(curr.right);
        }
  
        return res;
    }
```
