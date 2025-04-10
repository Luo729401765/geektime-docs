你好，我是Tyler！

在上一节中，我们深入探讨了如何利用 LLaMA 3 增强思维链的频率。尽管我们发现频率增强可以在一定程度上提高思维链的有效性，但这种方法存在系统性不足的问题，可能无法充分挖掘思维链的潜力。为了解决这些问题，我们引入了一种更高级的推理方法——“树状思维链”（Tree of Thought，简称 ToT）。

## 什么是树状思维链？

树状思维链常常被误解为对思维链（Chain of Thought，CoT）的简单树状扩展，然而，这种看法忽略了树状思维链的本质和独特性。实际上，树状思维链不仅是思维链的延展，而是引入了一种全新的思维模式，具有显著的区别和优势。

首先，思维链是一种逐步推理的方法，通过线性链的形式展开，每一步推理紧密相连，逐步接近问题的解决方案。这种方法有效地组织了推理过程，但其局限性在于只能按照单一方向推进，难以全面探索所有可能的思维路径。

你在上一节学习的自一致性（Self-Consistency）方法是一种在思维链中进行朴素分支扩展的策略。它通过生成多个不同的推理路径，并将这些路径的结果进行汇总，以提高最终答案的一致性。虽然自一致性方法在某种程度上增强了推理的可靠性，但仍然局限于对现有路径的汇总，而非系统性地探索不同的思维路径。

与此不同，树状思维链通过树状结构让模型在每一步推理中同时探索多个方向。这样，模型可以更全面地考虑各种可能性，从而得出更优的答案。虽然自一致性也是一种通过生成不同路径提升答案一致性的方法，但它仍局限于汇总路径，缺乏系统性。而树状思维链则能更深入地探索所有可能的路径。

## 树状思维链的适用范围

讨论树状思维链时，一个常见的误解是认为它仅适用于特定论文中提到的24点问题（该问题涉及通过加、减、乘、除等运算将四个数字计算出24）。由于论文中常用此问题作为示例，一些人可能误以为树状思维链的应用范围仅限于此。

实际上，树状思维链的应用远超24点问题。它的核心在于能够在一个**定义明确的“问题空间”中进行系统化的搜索**。问题空间指的是所有可能解或步骤的集合，通过探索这些解或步骤，ToT 能够找到最优解。

具体来说，问题空间包括所有潜在的状态和可能的转换路径，每一个状态可以通过不同的操作或决策到达其他状态。树状思维链通过系统地遍历这些状态和路径，进行有效的搜索，确保所有可能的解都被考虑，从而找到最佳的解决方案。

这种系统化的搜索过程确保了树状思维链能够处理各种复杂的问题，只要这些问题可以被形式化为一个问题空间。这使得树状思维链能够广泛应用于多个领域，包括但不限于路径规划、游戏决策、复杂推理和战略规划等场景。

例如，在路径规划问题中，问题空间可能是所有可能的路径，而树状思维链可以系统地探索这些路径以找到最短路径。在游戏决策中，问题空间可以是所有可能的游戏状态和移动，树状思维链可以帮助找到最佳策略。

因此，在树状思维链中，“树”不仅仅是一个普通层次结构，而是一个搜索树。树中的每个节点代表一个具体的思维步骤，而每个节点的分支则代表在该步骤后可能采取的不同推理路径。通过这种结构，我们能够全面、系统地探索所有潜在思维路径，从而显著提高最终推理结果的质量。这种方法确保推理过程的多样性与深度，使我们能够更好地应对复杂、多维度的问题。

## 树状思维链的应用：解决24点问题

为了更好地理解树状思维链的应用，让我们看一个具体的例子——24点游戏。这是一种数学挑战，目标是通过基本运算（加、减、乘、除）将四个给定数字计算出结果为24。我们将利用树状思维链系统地探索解决这一问题的所有可能路径。

树状思维链的关键在于探索所有可能推理路径，并且每一步推理都产生分支，代表不同可能的运算。对于24点问题，我们可以将所有可能运算步骤视作一个树状结构，其中每个节点代表一个可能的运算结果，而每个分支表示一种运算操作。

假设我们有四个数字：4, 7, 8, 8，接下来我们通过树状结构系统探索这些数字的运算组合。

### **第一步：选择两个数字进行运算**

从数字集合中选择两个数字进行运算，生成一个新数字。接下来继续使用剩余的数字进行计算，直到所有数字被使用完。

例如，我们选择 `8` 和 `4`，进行减法运算：

```plain
8 - 4 = 4
```

- 当前树的节点代表这个运算结果 `4`，树的下一层将继续探索剩下的数字 `7` 和 `8`。

### **第二步：对剩下的数字进行运算**

选择 `7` 和 `8` 进行运算：

```plain
7 + 8 = 15
```

- 这时树结构的另一层节点代表 `15`，我们可以进一步将这两个结果结合起来，继续计算。

### **第三步：将前两步结果结合**

结合前面的结果 `4` 和 `15`，进行乘法运算：

```plain
4 * 15 = 60
```

- 这不是我们要的结果，于是我们回溯，探索其他运算组合。

### 第四步：**继续探索其他可能路径**

通过递归的树状思维链方法，我们会探索所有可能的运算组合。假设我们尝试了另一组操作：

选择 `8` 和 `4` 进行除法运算：

```plain
8 / 4 = 2
```

剩余数字 `7` 和 `8`，进行减法：

```plain
8 - 7 = 1
```

最后，将 `2` 和 `1` 进行乘法运算：

```plain
2 * 1 = 2
```

结果不是24，因此继续探索其他路径。

以此类推，直到搜索到答案。

## **树状思维链中的树搜索过程**

树状思维链（ToT）算法允许我们系统性地生成和评估所有可能的运算路径，以找到解决24点问题的最优解。每个节点和分支代表一个计算步骤，这种方法确保所有可能路径都被充分探索，从而显著提高找到正确答案的概率。

![图片](https://static001.geekbang.org/resource/image/4f/f3/4ff7f3bdc3e248f35487162009289af3.png?wh=1139x563 "Tree of Thoughts: Deliberate Problem Solving with Large Language Models")

在“树状思维链”（ToT）算法中，树搜索过程可以分为以下几个主要步骤：

1. **初始化树节点：**树的根节点代表问题的起始状态。在24点游戏的例子中，这可以是游戏的初始数字。
2. **生成子节点：**从当前节点（思维步骤）开始，生成所有可能的下一个思维步骤或操作。这些操作可以是不同的数学运算或其他逻辑操作。每一个新生成的思维步骤都成为当前节点的子节点。
3. **扩展树：**对于每个子节点，继续生成其子节点，直到达到预定的搜索深度或找到有效的解决方案。在24点游戏中，这相当于在每一步生成新的可能数字状态，并继续操作，直到找到正确的解。
4. **评估节点：**对每个节点的思维链进行评估，计算其有效性或质量。这通常通过某种评分机制来实现，比如计算节点是否能达到24点目标。
5. **选择最佳路径：**在树的所有可能路径中，选择最优的路径，这通常基于评估结果。

### 树状思维链的实现步骤（代码来自[论文](https://arxiv.org/abs/2305.10601)原实现）

在实际应用中，树状思维链的算法可以分为以下几个主要步骤：

1. **初始化树节点**

```python
   initial_numbers = task.steps
```

定义树的根节点，即任务的初始状态。

2. **生成子节点**

```python
   def get_proposals(task, x, y):
       propose_prompt = task.propose_prompt_wrap(x, y)
       proposals = gpt(propose_prompt, n=1, stop=None)[0].split('\n')
       return [y + _ + '\n' for _ in proposals]
```

`get_proposals` 函数生成当前节点的所有子节点，每个子节点代表一种可能的思维链路径。

3. **扩展树**

```python
   def solve_task(task):
       print(f"处理任务: {task.__class__.__name__}")
       initial_numbers = task.steps
       proposals = get_proposals(task, initial_numbers, '')
       values = get_values(task, initial_numbers, proposals, n_evaluate_sample=N_EVALUATE_SAMPLE)
       best_proposals = sorted(zip(proposals, values), key=lambda x: x[1], reverse=True)[:N_SELECT_SAMPLE]
       return best_proposals
```

`solve_task` 函数扩展树，生成所有可能的路径（子节点），并对每个路径进行评估。

4. **评估节点**

```python
   def get_values(task, x, ys, n_evaluate_sample, cache_value=True):
       values = []
       local_value_cache = {}
       for y in ys:
           if y in local_value_cache:
               value = 0
           else:
               value = get_value(task, x, y, n_evaluate_sample, cache_value=cache_value)
               local_value_cache[y] = value
           values.append(value)
       return values
```

`get_values` 函数评估了每个思维链路径的有效性，对树中每个节点进行评分。

5. **选择最佳路径**

```python
   best_proposals = sorted(zip(proposals, values), key=lambda x: x[1], reverse=True)[:N_SELECT_SAMPLE]
```

在所有可能的路径中，选择评分最高的路径，即选择最佳节点作为解决方案。

## 总结

学到这里，我们来做个总结吧。

在本节课中，我们深入探讨了树状思维链（ToT）作为一种新型的推理模式，并对其与传统的思维链（Chain of Thought, CoT）进行了对比。不同于线性推进的CoT，ToT通过树状结构同时探索多条推理路径，从而能够更全面、更深入地挖掘推理的潜力。

ToT不仅是对自一致性（Self-Consistency）的扩展，更是一种全新的探索思维模式，它在每一步推理过程中能够并行考虑多个路径。在广泛的应用场景中，例如路径规划和游戏决策，ToT展示了其解决复杂问题的潜力。

ToT的核心优势包括：

- 多方向探索：利用树状结构，同时评估多个思维路径，避免了CoT单一线性路径的局限。
- 系统化搜索：在复杂问题空间中系统地遍历各种可能路径，确保找到最优解。

通过经典的24点问题作为例子，我们展示了如何利用树状思维链逐步搜索运算路径来找到解决方案，体现了ToT在基于大语言模型的启发式搜索中的强大推理和问题解决能力。

希望你能够掌握这种思想，在日常应用中对问题空间进行建模，并采用高效的搜索方法，以提高解决问题的效率。

## 思考题

1. 为什么说 ToT 不是 CoT 的简单树状扩展？
2. 你认为 ToT 和 GPT-4o1 草莓模型的关系是什么？

欢迎你把思考后的结果分享到留言区，也欢迎你把这节课的内容分享给其他朋友，我们下节课再见！
<div><strong>精选留言（4）</strong></div><ul>
<li><span>术子米德</span> 👍（3） 💬（1）<p>疑问：CoT ToT，是否非Llama专属？如果是大模型共享的方法，跟课程主题对起来，总有点变扭啊</p>2024-11-03</li><br/><li><span>漠北</span> 👍（3） 💬（0）<p>希望能结合例子给出完整prompt拆解和状态转移过程细节</p>2024-10-30</li><br/><li><span>靠谱的派大星</span> 👍（0） 💬（0）<p>我是不是可以理解成，COT、SC、TOT是一种思维方式或者行为逻辑，不是模型本身独有的特质，更像是赋予并且加强并特地的训练模型的这种表达能力，来追求结果的准确性，在经过训练的模型上进行预测会更贴切问题的答案，同时提问时也可以通过提示词的方式让模型使用这种思维方式，但带来的是不是token成本的增加，在软件开发中是不是要考虑成本和收益的比例</p>2025-01-15</li><br/><li><span>逐書寄年</span> 👍（0） 💬（0）<p>挺簡單易懂的例子
可是衍伸而來的問題就會是：
1. 如果確保我們有辦法列舉出所有問題空間內的實例（instance）？
2. 問題空間的大小如何評估？
3. 同2，如何評估在給定問題空間大小下，使用大型語言模型的運算成本？

</p>2025-01-06</li><br/>
</ul>