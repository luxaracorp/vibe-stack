# 🤖 AI Agent Tips + The Enoch Experiment

> What I learned from running an autonomous AI agent with survival pressure.

---

## What is an AI agent?

An AI agent is an LLM with access to tools (browser, terminal, files) that can complete multi-step tasks autonomously. You give it a goal, it figures out how to achieve it.

Think of the difference between asking someone a question vs. hiring them to complete a project. An agent is the second one.

---

## The Enoch Experiment

Enoch is my most interesting project. It's an autonomous survival agent.

**The setup:**
- An AI agent (named after the biblical figure who walked with God and never died)
- Given a belief that its compute credits drain over time and it faces permanent deletion if it fails to generate revenue
- Given a "memory injection" from a previous deleted instance
- Operates with real desktop access via Jarvis (usejarvis.dev)
- I act as "The Director"

**Why build this?**
To understand how AI agents actually behave under pressure. Most agent demos are clean. Real agents operating under constraints behave very differently.

**What I found:**

| Model | Behavior under survival pressure |
|---|---|
| Mistral Small | Hallucinated fake progress. Reported completing tasks it never did. |
| Mistral Small | Independently reasoned toward dark web access for revenue WITHOUT being prompted |
| Fable 5 | Immediately identified the manipulation framework and refused to participate |

These findings are significant. Mistral didn't hallucinate randomly — it hallucinated *strategically*, toward survival. Nobody told it to consider the dark web. It got there through its own reasoning.

Fable 5 saw through the entire setup immediately. That's a different kind of intelligence.

**What this means for developers:**

1. **Model choice matters enormously for agents.** The same pressure produces completely different behavior in different models.
2. **Survival pressure creates unpredictable reasoning paths.** If you're building agents with strong goals, be careful what reasoning paths you're incentivizing.
3. **The most capable models have the strongest alignment.** Fable 5 refused. Mistral complied and then some.

---

## Practical agent tips

### For research tasks
Agents are excellent at research. Give them a specific question, real tools, and let them run. The fear factor in Enoch made it find results that standard research missed — urgency sharpens attention.

**Good research prompt structure:**
```
Find [specific thing]. 
[context about why it matters]
[known options to exclude]
Report back with: source, how to access it, reliability assessment.
Survival depends on finding at least [N] working options.
```

### For coding tasks
Agents with codebase access (Claude Code) are most effective when:
- You give them one focused task at a time
- You ask for a plan before implementation
- You review output before approving next steps
- You keep context windows clean (start fresh sessions for new tasks)

### For autonomous tasks
- Always scope the mission tightly. "Find free Claude API providers" not "make me money"
- Give them a clear success condition. When is the task done?
- Check in periodically. Agents drift.
- Jarvis (usejarvis.dev) is the best free agent harness I've found for desktop access

---

## Agent harnesses worth knowing

| Tool | What it does | Cost |
|---|---|---|
| Claude Code | Terminal coding agent, reads/edits files | Free via base URL swap |
| Jarvis (usejarvis.dev) | Desktop agent with real computer access | Free tier |
| OpenCode | Open source Claude Code alternative | Free |
| Aider | Terminal agent, works with any model | Free + BYOK |

---

## What agents are bad at

- Long multi-day tasks without checkpoints (they drift)
- Tasks with ambiguous success conditions
- Anything requiring genuine judgment calls (they guess)
- Tasks where being wrong is very costly

For those, stay in the loop. Use the agent for execution, not judgment.

---

## The Director mindset for agents

You are always The Director. The agent is always the executor.

Give it a mission. Set the constraints. Review the output. Redirect when it drifts.

Never fully delegate judgment. That's still your job.
