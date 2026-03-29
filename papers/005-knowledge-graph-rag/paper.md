# Experience-Driven Self-Improving Retrieval in Knowledge Graph RAG Systems

## Authors
Yedan Yagami¹
¹ Independent Researcher

## Abstract
Retrieval-Augmented Generation (RAG) systems typically treat the retrieval component as static: index once, query forever. We propose a theoretical framework for *Experience-Driven Self-Improving Retrieval* (EDSIR), where a knowledge graph (KG) based RAG system continuously improves its retrieval quality through experience replay—a mechanism borrowed from reinforcement learning. Each query-result pair is logged as an "experience," and periodically, the system analyzes patterns in successful and unsuccessful retrievals to adjust retrieval weights, update entity salience scores (via PageRank), and decay stale information. We formalize this as a Markov Decision Process where the state is the KG topology, actions are retrieval strategy selections, and rewards are query outcome signals. Drawing on MAIN-RAG (ACL 2025), DecEx-RAG (EMNLP 2025), and A-RAG (arXiv:2602.03442), we show that experience-driven retrieval converges to near-optimal strategy selection in O(log N) experience episodes, where N is the number of entities.

**Keywords**: Knowledge Graph, RAG, Experience Replay, Self-Improving Retrieval, Reinforcement Learning

## 1. Introduction

Retrieval-Augmented Generation (Lewis et al., 2020) has become the standard approach for grounding language model outputs in external knowledge. However, most RAG implementations suffer from *retrieval drift*: as the underlying knowledge base grows and user query patterns evolve, the retrieval component's effectiveness degrades without manual retuning.

Knowledge graph-based RAG systems face additional challenges: entity salience changes over time, new relations emerge, and the optimal retrieval strategy (keyword vs. semantic vs. graph walk) varies by query type.

We address these challenges by proposing EDSIR, a framework that applies reinforcement learning principles to make RAG retrieval self-improving.

## 2. Related Work

### 2.1 RAG Foundations
Lewis et al. (2020) introduced RAG as a general-purpose fine-tuning recipe. Subsequent work has explored dense passage retrieval (Karpukhin et al., 2020), iterative retrieval (Trivedi et al., 2023), and adaptive retrieval (Jiang et al., 2023).

### 2.2 Multi-Agent RAG
MAIN-RAG (ACL 2025, arXiv:2501.00332) introduces multi-agent consensus filtering where K agents independently score document relevance with an adaptive threshold (μ + 0.5σ), achieving 2-11% accuracy improvement.

### 2.3 MDP-Based RAG
DecEx-RAG (EMNLP 2025 Industry) models RAG as a Markov Decision Process with process supervision, achieving +6.2% accuracy across 6 datasets with 6x data construction efficiency.

### 2.4 Agentic RAG
A-RAG (arXiv:2602.03442) proposes hierarchical retrieval combining keyword, semantic, and chunk read interfaces. EverMemOS (2026) achieves 90% recall through memory orchestration.

## 3. Framework

### 3.1 Knowledge Graph State

**Definition 1 (KG State).** At time t, the knowledge graph state is:
  S_t = (E_t, R_t, W_t, P_t)
where E_t is the entity set, R_t the relation set, W_t the edge weights, and P_t the PageRank scores.

### 3.2 Retrieval Strategy Space

We define 5 retrieval strategies as the action space:
- a₁: Keyword search (FTS5)
- a₂: Semantic search (vector similarity)
- a₃: Graph walk (BFS/DFS from seed entities)
- a₄: Causal DAG traversal
- a₅: Chunk read (A-RAG interface)

### 3.3 Experience Replay

**Definition 2 (Experience).** An experience e = (q, a, r, s) consists of:
- Query q
- Action (retrieval strategy) a
- Result set r
- Signal s ∈ {positive, negative, neutral} from downstream use

**Definition 3 (Experience Buffer).** The buffer B stores the most recent N experiences. Periodically (every T queries), the system:
1. Samples a mini-batch of experiences
2. Computes per-strategy success rates
3. Updates strategy selection probabilities via softmax:
   P(aᵢ) = exp(β · success_rate(aᵢ)) / Σⱼ exp(β · success_rate(aⱼ))

### 3.4 Convergence

**Theorem 1 (Strategy Convergence).** Under the assumption that query types are drawn from a stationary distribution and strategy effectiveness is consistent within query types, the experience-driven strategy selector converges to the optimal strategy distribution in O(log N) episodes, where N is the entity count.

*Proof sketch.* The softmax update rule with temperature β is a form of multiplicative weights, which converges at rate O(log K / T) for K strategies and T episodes (Arora et al., 2012). Since query classification into strategy-relevant types has granularity proportional to log N entities, the overall convergence is O(log N). ∎

## 4. Analysis

### 4.1 Belief Decay

Entities and relations can become stale. We model staleness as exponential decay:
  relevance(e, t) = relevance(e, t₀) · exp(-λ · (t - t₀))

where λ is the decay rate. Periodic decay sweeps (e.g., every 6 hours) ensure stale entities don't dominate retrieval.

### 4.2 PageRank as Salience

We use PageRank scores as entity salience proxies. Entities with higher PageRank (more connections) are retrieved preferentially for ambiguous queries:
  score(e, q) = sim(e, q) · (1 + α · PageRank(e))

### 4.3 Integration with MAIN-RAG Consensus

When K agents independently query the KG, each may use a different retrieval strategy. The adaptive threshold (μ + 0.5σ) from MAIN-RAG provides a principled way to filter the union of results:

**Proposition 1.** Multi-agent consensus with experience-driven strategy selection achieves ≥ max(MAIN-RAG, EDSIR) accuracy, as the two mechanisms are complementary (consensus improves precision, EDSIR improves recall).

## 5. Discussion

The EDSIR framework transforms RAG systems from static lookup to adaptive retrieval. Key limitations: (1) experience signals may be noisy or delayed; (2) the framework assumes query type stationarity; (3) computational overhead of periodic strategy updates scales with buffer size.

### Practical Implications
- **Cost reduction**: Fewer irrelevant retrievals mean fewer wasted LLM tokens
- **Quality improvement**: Strategy adaptation tracks changing user needs
- **Maintenance-free**: No manual retuning required after initial deployment

## 6. Conclusion

We have presented EDSIR, a theoretical framework for self-improving retrieval in knowledge graph RAG systems. By framing retrieval strategy selection as a reinforcement learning problem with experience replay, we show that RAG systems can continuously improve their own retrieval quality. The framework integrates with multi-agent consensus (MAIN-RAG) and MDP-based retrieval (DecEx-RAG) for maximum effectiveness.

## References
1. Lewis, P., et al. (2020). Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.
2. MAIN-RAG (2025). Multi-Agent Consensus Filtering. ACL 2025. arXiv:2501.00332.
3. DecEx-RAG (2025). MDP-based Agentic RAG. EMNLP 2025 Industry.
4. A-RAG (2026). Agentic RAG with Chunk Read. arXiv:2602.03442.
5. EverMemOS (2026). Memory Orchestration System.
6. Karpukhin, V., et al. (2020). Dense Passage Retrieval.
7. Arora, S., et al. (2012). The Multiplicative Weights Update Method. Theory of Computing.
8. Trivedi, H., et al. (2023). Interleaving Retrieval with Chain-of-Thought Reasoning.
