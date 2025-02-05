# TreeOfThought
TOT Implementation
Have written a medium article on this: https://medium.com/@shekharsomani98/tree-of-thoughts-advancing-language-model-problem-solving-through-strategic-exploration-and-643776a03c40
Input-Output (IO) Prompting:
Think of IO prompting as having a simple conversation - you ask one question and get one answer. It's the most straightforward way to interact with AI models.
When to Use
Quick factual queries
Simple translations
Basic creative tasks
Single-step instructions
# Example IO Prompt
prompt = "What's the capital of France?"
# Direct Response: "Paris"

Chain-of-Thought (CoT) Prompting:
CoT prompting is like asking someone to show their work rather than just providing the final answer. It breaks down complex reasoning into clear, logical steps.
Key Components
Present the problem
Request step-by-step reasoning
Get detailed explanation with final answer
prompt = """
Calculate the total cost of 3 books at $12.99 with 8% tax.
Let's solve this step by step:
"""
# Response includes reasoning steps and final answer

Use IO prompting for straightforward tasks where you need quick answers. Switch to CoT when dealing with complex problems requiring detailed reasoning or when transparency in the thinking process is important.
Remember: While CoT provides more detailed reasoning, it requires larger models and may introduce unnecessary complexity for simple tasks. Choose your prompting method based on your specific needs and the complexity of your task.
Tree-of-Thought(ToT) prompting:
Tree of Thought (ToT) is an advanced evolution of Chain of Thought prompting that explores multiple reasoning paths simultaneously, similar to how a chess player considers various possible moves. Unlike linear problem-solving methods, ToT creates a hierarchical structure where multiple reasoning paths are explored simultaneously. The system generates and maintains various thought states as intermediate steps, organizing them in a tree-like structure with branches representing different solution paths.
The solving process uses a breadth-first search approach:
• Generates multiple thoughts at each step
• Evaluates each thought path
• Selects the best paths to explore further
• Continues until reaching a final answer or maximum steps
Code WalkThrough:
Unlike Chain of Thought which follows a single linear path, ToT creates a tree-like structure where multiple thoughts are generated and evaluated at each step. Here’s how it works:
Thought Generation: The system generates multiple potential reasoning paths at each step added to this class
def generate_thoughts(self, question: str, current_thought: str = "", n_samples: int = 3) -> List[str]:
       """Generate multiple possible next steps in reasoning."""
       # [IMPORTANT] A one-shot example can help the model follow your instructions precisely.
       # TODO: Implement thought generation function
       example_prompt = """Example:
           Question: If Jane has 3 apples and gives 2 to Bob, how many does she have?
           Current thought:
           Possible next steps:
           1. Subtract the 2 apples given away from the initial 3: 3 - 2 = 1.
           2. Check if there are any other apples involved; since none, the answer is 1.
           3. Verify by adding Bob's apples to Jane's remaining: 1 + 2 = 3, which matches the initial count.
           """
       prompt = f"""You are a logical thinker and math problem solver. Given a question and current thought, list {n_samples} possible next steps to solve the problem. Each step must be a concise sentence. Follow the example format.
           {example_prompt}
           Question: {question}
           Current thought: {current_thought}
           Possible next steps:
           """
       response = self.chat_with_gpt(prompt, n=1, stop=["\n\n"])[0]
       thoughts = []
       for line in response.split('\n'):
           line = line.strip()
           if line:
               thought = re.sub(r'^[\d\-*]+\s*\.?\s*', '', line).strip()
               if thought:
                   thoughts.append(thought)
       return thoughts[:n_samples]



Evaluation and Selection: Unlike CoT, ToT actively evaluates each thought path and selects the most promising ones.
   def evaluate_thought(self, question: str, thought: str, cache: bool = True) -> float:
       prompt = (
           f"Question: {question}\n"
           f"Partial reasoning:\n{thought}\n\n"
           f"Use the above Partial Reasoning to Rate from 1 to 5 how likely this path is correct. so that we can move with that path, if you think you got the \
               correct answer end the reasoning and provide the final answer only, "
       )
       eval_responses = self.chat_with_gpt(prompt, n=1)
       eval_text = eval_responses[0]
       match = re.search(r'([1-5])', eval_text)
       if match:
           score = float(match.group(1))
       else:
           score = 1.0  # fallback if parsing fails
       return score


   def select_best_thoughts(self, thoughts: List[str], scores: List[float], k: int = 2) -> List[str]:
       """Select top-k thoughts based on evaluation scores."""
       combined = sorted(zip(thoughts, scores), key=lambda x: x[1], reverse=True)
       return [thought for thought, score in combined[:k]]

BFS Implementation:
   def solve(self, question: str, max_steps: int = 8, n_samples_per_step: int = 3,
            k_best_thoughts: int = 2) -> str:
       """Solve a problem using Tree-of-Thoughts BFS."""
       current_thoughts = [""]
       final_answer = ""
       for step in range(max_steps):
           new_thoughts = []
           scores = []
           for thought in current_thoughts:
               # Generate new thoughts
               candidates = self.generate_thoughts(question, thought, n_samples_per_step)
               if not candidates:
                   continue
               # Evaluate candidates
               candidate_scores = [self.evaluate_thought(question, f"{thought} {c}".strip())for c in candidates]
               # Select and accumulate best candidates
               best_candidates = self.select_best_thoughts(candidates, candidate_scores, k_best_thoughts)
               new_thoughts.extend([f"{thought} {c}".strip() for c in best_candidates])
               scores.extend(candidate_scores[:k_best_thoughts])
           if not new_thoughts:
               break
           # Select best thoughts for next iteration
           current_thoughts = self.select_best_thoughts(new_thoughts, scores, k_best_thoughts)
           # Early termination if final answer detected
           for t in current_thoughts:
               if "final answer" in t.lower():
                   final_answer = t
                   prompt = f"""Get the final numeric answer to the question : {question}
                   Using the current logic: {final_answer}
                   Important : Your Answer should end with the number which is the final answer to the question asked. """
                   response = self.chat_with_gpt(prompt, n=1)
                   return final_answer.split("final answer")[-1].strip() + "final answer is" + str(response[0])
       prompt = f"""Get the final numeric answer to the question : {question}
       Using the current logic: {str(current_thoughts[0])}
       Important : Your Answer should end with the number which is the final answer to the question asked. """
       response = self.chat_with_gpt(prompt, n=1)
       return current_thoughts[0]+ " answer is " +str(response[0]) if current_thoughts else ""



Benefits and Applications of Tree of Thoughts (ToT):
Performance Boost:
Achieves 74% success rate in Game of 24 (compared to 4% with standard methods)
Mirrors human cognitive processes through structured problem decomposition
Enables backtracking and global decision optimization
Optimal Use Cases:
Strategic planning requiring multiple path exploration
Complex mathematical reasoning
Creative writing with structured narratives
Multi-variable optimization problems
Decision-making with interconnected factors
Implementation Considerations
Resource Requirements
High computational power needed
Substantial memory allocation for path maintenance
Complex system integration
Technical Limitations
Resource-intensive processing
Complex implementation requirements
Risk of over-focusing on specific reasoning branches
Requires careful component configuration
The framework excels in scenarios where traditional linear approaches fall short, particularly in problems requiring deep exploration and structured decision-making. However, its resource-intensive nature means implementation should be carefully considered based on available computational resources and problem complexity.
