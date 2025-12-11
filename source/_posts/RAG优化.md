---
title: RAG优化
date: 2025-4-21 16:10:12
description: 总结了一下当前RAG常见的优化方法
cover: /covers/22.png
tags:
    - Rag
---

### 1. 基本的RAG流程-Native RAG

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/2.png" alt="2" style="zoom:33%;" />

### 2. RAG优化

#### 2.1 优化chunk

传统的RAG在数据分块时采用的是固定分块，因为可能会破坏段落的连续性，因此这一方面可以改进

##### 2.1.1 拼接断裂的块

- 将固定分块的断裂部分找到，然后拼接起来；

```Python
def compute_breakpoints(similarities, method="percentile", threshold=90):
    """
    Computes chunking breakpoints based on similarity drops.

    Args:
    similarities (List[float]): List of similarity scores between sentences.
    method (str): 'percentile', 'standard_deviation', or 'interquartile'.
    threshold (float): Threshold value (percentile for 'percentile', std devs for 'standard_deviation').

    Returns:
    List[int]: Indices where chunk splits should occur.
    """
    # Determine the threshold value based on the selected method
    if method == "percentile":
        # Calculate the Xth percentile of the similarity scores
        threshold_value = np.percentile(similarities, threshold)
    elif method == "standard_deviation":
        # Calculate the mean and standard deviation of the similarity scores
        mean = np.mean(similarities)
        std_dev = np.std(similarities)
        # Set the threshold value to mean minus X standard deviations
        threshold_value = mean - (threshold * std_dev)
    elif method == "interquartile":
        # Calculate the first and third quartiles (Q1 and Q3)
        q1, q3 = np.percentile(similarities, [25, 75])
        # Set the threshold value using the IQR rule for outliers
        threshold_value = q1 - 1.5 * (q3 - q1)
    else:
        # Raise an error if an invalid method is provided
        raise ValueError("Invalid method. Choose 'percentile', 'standard_deviation', or 'interquartile'.")

    # Identify indices where similarity drops below the threshold value
    return [i for i, sim in enumerate(similarities) if sim < threshold_value]
```

##### 2.1.2 使用不同的分块大小

- (例如原来是256， 现在试一试512， 1000等等)；

##### 2.1.3 总结chunk

- chunk Embedding之后缺乏语义信息，在chunk Embedding之前 给每一个chunk添加一个meta data, 一般是对这段text的总结，如标题等(可以使用LLM来协助),  在查询时， 使用的是两个信息相似度的平均值 （这个demo）

```Python
system_prompt = "Generate a concise and informative title for the given text."

# **************************** step 2 *********************************
chunks = []  # Initialize an empty list to store chunks

# Iterate through the text with the specified chunk size and overlap
for i in range(0, len(text), n - overlap):
    chunk = text[i:i + n]  # Extract a chunk of text
    header = generate_chunk_header(chunk)  # Generate a header for the chunk using LLM
    chunks.append({"header": header, "text": chunk})  # Append the header and chunk to the list

return chunks  # Return the list of chunks with headers

# *************************** step 3 **********************************
for chunk in tqdm(text_chunks, desc="Generating embeddings"):
    # Create an embedding for the chunk's text
    text_embedding = create_embeddings(chunk["text"])
    # Create an embedding for the chunk's header
    header_embedding = create_embeddings(chunk["header"])
    # Append the chunk's header, text, and their embeddings to the list
    embeddings.append({"header": chunk["header"], "text": chunk["text"], "embedding": text_embedding, "header_embedding": header_embedding})
    
    
# ****************************** step 4**************************************

# Iterate through each chunk to calculate similarity scores
for chunk in chunks:
    # Compute cosine similarity between query embedding and chunk text embedding
    sim_text = cosine_similarity(np.array(query_embedding), np.array(chunk["embedding"]))
    # Compute cosine similarity between query embedding and chunk header embedding
    sim_header = cosine_similarity(np.array(query_embedding), np.array(chunk["header_embedding"]))
    # Calculate the average similarity score
    avg_similarity = (sim_text + sim_header) / 2
    # Append the chunk and its average similarity score to the list
    similarities.append((chunk, avg_similarity))
```

##### 2.1.4 提问chunk

- 同上，不过这里改变了方向，是从chunk中提取问题，而不是总结，换汤不换药， 注意这里的是 问题 + chunk text 向量化之后一起 进行检索，然后找出top-k

```python
# Define the system prompt to guide the AI's behavior
system_prompt = "You are an expert at generating relevant questions from text. Create concise questions that can be answered using only the provided text. Focus on key information and concepts."

# Define the user prompt with the text chunk and the number of questions to generate
user_prompt = f"""
Based on the following text, generate {num_questions} different questions that can be answered using only this text:

{text_chunk}

Format your response as a numbered list of questions only, with no additional text.
"""
```

##### 2.1.5 提取chunk，不采用固定chunk size

- **Proposition Chunking**

仅仅采用固定分块，问题太多，会破坏句子的语义，句子不完整时会产生歧义等, 因此这种方法是从每个chunk中提取有用的关键信息作为新的chunk (这里也是使用LLM来分句的)

主要是两步：分段 然后评估筛选出最符合的

```Python
############### 产生新的chunk
# System prompt to instruct the AI on how to generate propositions
system_prompt = """Please break down the following text into simple, self-contained propositions. 
Ensure that each proposition meets the following criteria:

1. Express a Single Fact: Each proposition should state one specific fact or claim.
2. Be Understandable Without Context: The proposition should be self-contained, meaning it can be understood without needing additional context.
3. Use Full Names, Not Pronouns: Avoid pronouns or ambiguous references; use full entity names.
4. Include Relevant Dates/Qualifiers: If applicable, include necessary dates, times, and qualifiers to make the fact precise.
5. Contain One Subject-Predicate Relationship: Focus on a single subject and its corresponding action or attribute, without conjunctions or multiple clauses.

Output ONLY the list of propositions without any additional text or explanations."""

# User prompt containing the text chunk to be converted into propositions
user_prompt = f"Text to convert into propositions:\n\n{chunk['text']}"


################# 评估新chunk的质量 (结合原始的chunk)
# System prompt to instruct the AI on how to evaluate the proposition
system_prompt = """You are an expert at evaluating the quality of propositions extracted from text.
Rate the given proposition on the following criteria (scale 1-10):

- Accuracy: How well the proposition reflects information in the original text
- Clarity: How easy it is to understand the proposition without additional context
- Completeness: Whether the proposition includes necessary details (dates, qualifiers, etc.)
- Conciseness: Whether the proposition is concise without losing important information

The response must be in valid JSON format with numerical scores for each criterion:
{"accuracy": X, "clarity": X, "completeness": X, "conciseness": X}
"""

# User prompt containing the proposition and the original text
user_prompt = f"""Proposition: {proposition}
```

#### 2.2 优化处理过程

##### 2.2.1 分层RAG

传统的RAG当 text很大时，chunk就很多，匹配chunk缺乏上下文，而且每一个chunk都做相似度计算，计算量大，效率低，因此分层RAG的思想是，先对每一页做一个总结，先在总结中找到相关信息，然后再在对应的页中去找相关的详细信息

```Python
############## 总结的prompt
# Define the system prompt to instruct the summarization model
system_prompt = """You are an expert summarization system.
Create a detailed summary of the provided text. 
Focus on capturing the main topics, key information, and important facts.
Your summary should be comprehensive enough to understand what the page contains
but more concise than the original."""
```

##### 2.2.2 GraphRAG

可参考：[GraphRAG快速入门与原理详解](https://zhuanlan.zhihu.com/p/13801755777)

![img](https://raw.githubusercontent.com/nashpan/image-hosting/main/asynccode)

- 实体提取 (Node, 节点)：利用LLM进行实体的提取，这些实体通常是指文档中出现的人物、地点、组织、概念等信息。
- 关系提取(Edage， 边)：关系挖掘是从文本中识别出实体之间的 关系，例如：谁与谁有关联、某个实体与另一个实体之间的关系是“属于”、“合作”、“对立”等。
- Community(中文翻译成社区，感觉真的很奇怪，直接使用英文就好) 构建： 感觉是为了将不同部分的信息联系起来 通过使用 图谱聚类算法（如 Leiden 算法），GraphRAG 会将不同的实体和关系分组，形成多个 社区（Community）。这些社区是根据实体之间的相似度或关系的密切程度进行划分的。这种分组帮助 GraphRAG 更好地理解不同知识领域的结构，

##### 2.2.3 KAG

https://github.com/OpenSPG/KAG/blob/master/README_cn.md

#### 2.3 后处理

由于固定分块会破坏句子的连续性

##### 2.3.1 增加相邻上下文

- 在检索这一步时，将检索到的相邻chunk也包含进来以增加上下文，减少信息的损失

##### 2.3.2 ReRank

- ReRank
  -  就是觉得通过计算余弦相似度的方式得到的相似度具有一定的可信度，但是不高，因此使用其它方法对得到的top-k进行重排序，然后选择相关度最高的几个答案(一般使用LLM来实现)

```python
system_prompt = """You are an expert at evaluating document relevance for search queries.
Your task is to rate documents on a scale from 0 to 10 based on how well they answer the given query.

Guidelines:
- Score 0-2: Document is completely irrelevant
- Score 3-5: Document has some relevant information but doesn't directly answer the query
- Score 6-8: Document is relevant and partially answers the query
- Score 9-10: Document is highly relevant and directly answers the query

You MUST respond with ONLY a single integer score between 0 and 10. Do not include ANY other text."""
```

也可以使用关键词的方式

```python
# Extract important keywords from the query
keywords = [word.lower() for word in query.split() if len(word) > 3]

scored_results = []  # Initialize a list to store scored results

for result in results:
    document_text = result["text"].lower()  # Convert document text to lowercase
    
    # Base score starts with vector similarity
    base_score = result["similarity"] * 0.5
    
    # Initialize keyword score
    keyword_score = 0
    for keyword in keywords:
        if keyword in document_text:
            # Add points for each keyword found
            keyword_score += 0.1
            
            # Add more points if keyword appears near the beginning
            first_position = document_text.find(keyword)
            if first_position < len(document_text) / 4:  # In the first quarter of the text
                keyword_score += 0.1
            
            # Add points for keyword frequency
            frequency = document_text.count(keyword)
            keyword_score += min(0.05 * frequency, 0.2)  # Cap at 0.2
    
    # Calculate the final score by combining base score and keyword score
    final_score = base_score + keyword_score
```

##### 2.3.3 增加上下文-v2

- 计算连续chunk (分段)的总相关性，然后输出 top-k 个分段

```python
def find_best_segments(chunk_values, max_segment_length=20, total_max_length=30, min_segment_value=0.2):
    """
    Find the best segments using a variant of the maximum sum subarray algorithm.
    
    Args:
        chunk_values (List[float]): Values for each chunk
        max_segment_length (int): Maximum length of a single segment
        total_max_length (int): Maximum total length across all segments
        min_segment_value (float): Minimum value for a segment to be considered
        
    Returns:
        List[Tuple[int, int]]: List of (start, end) indices for best segments
    """
    print("Finding optimal continuous text segments...")
    
    best_segments = []
    segment_scores = []
    total_included_chunks = 0
    
    # Keep finding segments until we hit our limits
    while total_included_chunks < total_max_length:
        best_score = min_segment_value  # Minimum threshold for a segment
        best_segment = None
        
        # Try each possible starting position
        for start in range(len(chunk_values)):
            # Skip if this start position is already in a selected segment
            if any(start >= s[0] and start < s[1] for s in best_segments):
                continue
                
            # Try each possible segment length
            for length in range(1, min(max_segment_length, len(chunk_values) - start) + 1):
                end = start + length
                
                # Skip if end position is already in a selected segment
                if any(end > s[0] and end <= s[1] for s in best_segments):
                    continue
                
                # Calculate segment value as sum of chunk values
                segment_value = sum(chunk_values[start:end])
                
                # Update best segment if this one is better
                if segment_value > best_score:
                    best_score = segment_value
                    best_segment = (start, end)
        
        # If we found a good segment, add it
        if best_segment:
            best_segments.append(best_segment)
            segment_scores.append(best_score)
            total_included_chunks += best_segment[1] - best_segment[0]
            print(f"Found segment {best_segment} with score {best_score:.4f}")
        else:
            # No more good segments to find
            break
    
    # Sort segments by their starting position for readability
    best_segments = sorted(best_segments, key=lambda x: x[0])
    
    return best_segments, segment_scores
```

##### 2.3.4 使用LLM过滤无效信息

- 使用LLM对top-k的chunk text进行判断，过滤，然后帮助我们筛选出最符合的信息，下面是实现的三种方式

```python
if compression_type == "selective":
    system_prompt = """You are an expert at information filtering. 
    Your task is to analyze a document chunk and extract ONLY the sentences or paragraphs that are directly relevant to the user's query. Remove all irrelevant content.

    Your output should:
    1. ONLY include text that helps answer the query
    2. Preserve the exact wording of relevant sentences (do not paraphrase)
    3. Maintain the original order of the text
    4. Include ALL relevant content, even if it seems redundant
    5. EXCLUDE any text that isn't relevant to the query

    Format your response as plain text with no additional comments."""
elif compression_type == "summary":
    system_prompt = """You are an expert at summarization. 
    Your task is to create a concise summary of the provided chunk that focuses ONLY on information relevant to the user's query.

    Your output should:
    1. Be brief but comprehensive regarding query-relevant information
    2. Focus exclusively on information related to the query
    3. Omit irrelevant details
    4. Be written in a neutral, factual tone

    Format your response as plain text with no additional comments."""
else:  # extraction
    system_prompt = """You are an expert at information extraction.
    Your task is to extract ONLY the exact sentences from the document chunk that contain information relevant to answering the user's query.

    Your output should:
    1. Include ONLY direct quotes of relevant sentences from the original text
    2. Preserve the original wording (do not modify the text)
    3. Include ONLY sentences that directly relate to the query
    4. Separate extracted sentences with newlines
    5. Do not add any commentary or additional text

    Format your response as plain text with no additional comments."""

# Define the user prompt with the query and document chunk
user_prompt = f"""
    Query: {query}

    Document Chunk:
    {chunk}

    Extract only the content relevant to answering this query.
"""
```

##### 2.3.5 优化检索策略- Adaptive Retrieval

本质还是query的改写

RAG的效果差，也有可能是检索的策略有问题，对于所有的问题，全部采用向量相似度匹配，可能不太对，因此针对不同的用户问题，需要采取不同的策略

- **对查询类型进行分类: 事实、分析、观点或情境  （使用大模型来进行选择）**
- 然后 Query Transform （根据上面的类型进行改写）, 然后和之前一样进行相似度进行匹配

```python
# Define the system prompt to guide the AI's classification
system_prompt = """You are an expert at classifying questions. 
    Classify the given query into exactly one of these categories:
    - Factual: Queries seeking specific, verifiable information.
    - Analytical: Queries requiring comprehensive analysis or explanation.
    - Opinion: Queries about subjective matters or seeking diverse viewpoints.
    - Contextual: Queries that depend on user-specific context.

    Return ONLY the category name, without any explanation or additional text.
"""
```

query 改写的部分

```Python
##################### 基于事实 Factual 
system_prompt = """You are an expert at enhancing search queries.
    Your task is to reformulate the given factual query to make it more precise and 
    specific for information retrieval. Focus on key entities and their relationships.

    Provide ONLY the enhanced query without any explanation.
"""

user_prompt = f"Enhance this factual query: {query}"

##################### 基于事实 Factual 
system_prompt = """You are an expert at enhancing search queries.
    Your task is to reformulate the given factual query to make it more precise and 
    specific for information retrieval. Focus on key entities and their relationships.

    Provide ONLY the enhanced query without any explanation.
"""

user_prompt = f"Enhance this factual query: {query}"

### 基于分析 Analytical 
# Define the system prompt to guide the AI in generating sub-questions
system_prompt = """You are an expert at breaking down complex questions.
Generate sub-questions that explore different aspects of the main analytical query.
These sub-questions should cover the breadth of the topic and help retrieve 
comprehensive information.

Return a list of exactly 3 sub-questions, one per line.
"""

# Create the user prompt with the main query
user_prompt = f"Generate sub-questions for this analytical query: {query}"


##################  基于分析 Analytical 
# Define the system prompt to guide the AI in generating sub-questions
system_prompt = """You are an expert at breaking down complex questions.
Generate sub-questions that explore different aspects of the main analytical query.
These sub-questions should cover the breadth of the topic and help retrieve 
comprehensive information.

Return a list of exactly 3 sub-questions, one per line.
"""

# Create the user prompt with the main query
user_prompt = f"Generate sub-questions for this analytical query: {query}"


################## 基于观点 Opinion Strategy  
# Define the system prompt to guide the AI in identifying different perspectives
system_prompt = """You are an expert at identifying different perspectives on a topic.
    For the given query about opinions or viewpoints, identify different perspectives 
    that people might have on this topic.

    Return a list of exactly 3 different viewpoint angles, one per line.
"""

# Create the user prompt with the main query
user_prompt = f"Identify different perspectives on: {query}"

# Extract and clean the viewpoints
viewpoints = response.choices[0].message.content.strip().split('\n')
viewpoints = [v.strip() for v in viewpoints if v.strip()]
print(f"Identified viewpoints: {viewpoints}")

# Retrieve documents representing each viewpoint
all_results = []
for viewpoint in viewpoints:
    # 注意这里的做法
    # Combine the main query with the viewpoint
    combined_query = f"{query} {viewpoint}"
    # Create embeddings for the combined query
    viewpoint_embedding = create_embeddings(combined_query)
    # Perform similarity search for the combined query
    results = vector_store.similarity_search(viewpoint_embedding, k=2)
    
    # Mark results with the viewpoint they represent
    for result in results:
        result["viewpoint"] = viewpoint
    
    # Add the results to the list of all results
    all_results.extend(results)
    
    
  ################### 基于情境  Contextual Strategy
if not user_context:
system_prompt = """You are an expert at understanding implied context in questions.
For the given query, infer what contextual information might be relevant or implied 
but not explicitly stated. Focus on what background would help answering this query.

Return a brief description of the implied context."""

user_prompt = f"Infer the implied context in this query: {query}"

# Generate the inferred context using the LLM
response = client.chat.completions.create(
    model="meta-llama/Llama-3.2-3B-Instruct",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt}
    ],
    temperature=0.1
)

# Extract and print the inferred context
user_context = response.choices[0].message.content.strip()
print(f"Inferred context: {user_context}")

# Reformulate the query to incorporate context
system_prompt = """You are an expert at reformulating questions with context.
Given a query and some contextual information, create a more specific query that 
incorporates the context to get more relevant information.

Return ONLY the reformulated query without explanation."""

user_prompt = f"""
Query: {query}
Context: {user_context}

Reformulate the query to incorporate this context:"""

# Generate the contextualized query using the LLM
response = client.chat.completions.create(
model="meta-llama/Llama-3.2-3B-Instruct",
messages=[
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": user_prompt}
],
temperature=0
)
  
```

##### 2.3.6 CRAG

**对检索的文件进行了修正**

校验检索得到的文档，如果文档相关性评分比较高(相关性分数还是使用LLM)，那么继续原先的步骤即可；如果相关性评分很低，那么采用外部的工具(如浏览器等)来搜索更多的资料，用以补充；如果相关性评分在中间，结合检索文本和搜索文本

```python
def crag_process(query, vector_store, k=3):
    print(f"\n=== Processing query with CRAG: {query} ===\n")
    
    # Step 1: Create query embedding and retrieve documents
    print("Retrieving initial documents...")
    query_embedding = create_embeddings(query)
    retrieved_docs = vector_store.similarity_search(query_embedding, k=k)
    
    # Step 2: Evaluate document relevance
    print("Evaluating document relevance...")
    relevance_scores = []
    for doc in retrieved_docs:
        score = evaluate_document_relevance(query, doc["text"])
        relevance_scores.append(score)
        doc["relevance"] = score
        print(f"Document scored {score:.2f} relevance")
    
    # Step 3: Determine action based on best relevance score
    max_score = max(relevance_scores) if relevance_scores else 0
    best_doc_idx = relevance_scores.index(max_score) if relevance_scores else -1
    
    # Track sources for attribution
    sources = []
    final_knowledge = ""
    
    # Step 4: Execute the appropriate knowledge acquisition strategy
    if max_score > 0.7:
        # Case 1: High relevance - Use document directly
        print(f"High relevance ({max_score:.2f}) - Using document directly")
        best_doc = retrieved_docs[best_doc_idx]["text"]
        final_knowledge = best_doc
        sources.append({
            "title": "Document",
            "url": ""
        })
        
    elif max_score < 0.3:
        # Case 2: Low relevance - Use web search
        print(f"Low relevance ({max_score:.2f}) - Performing web search")
        web_results, web_sources = perform_web_search(query)
        final_knowledge = refine_knowledge(web_results)
        sources.extend(web_sources)
        
    else:
        # Case 3: Medium relevance - Combine document with web search
        print(f"Medium relevance ({max_score:.2f}) - Combining document with web search")
        best_doc = retrieved_docs[best_doc_idx]["text"]
        refined_doc = refine_knowledge(best_doc)
        
        # Get web results
        web_results, web_sources = perform_web_search(query)
        refined_web = refine_knowledge(web_results)
        
        # Combine knowledge
        final_knowledge = f"From document:\n{refined_doc}\n\nFrom web search:\n{refined_web}"
        
        # Add sources
        sources.append({
            "title": "Document",
            "url": ""
        })
        sources.extend(web_sources)
    
    # Step 5: Generate final response
    print("Generating final response...")
    response = generate_response(query, final_knowledge, sources)
    
    # Return comprehensive results
    return {
        "query": query,
        "response": response,
        "retrieved_docs": retrieved_docs,
        "relevance_scores": relevance_scores,
        "max_relevance": max_score,
        "final_knowledge": final_knowledge,
        "sources": sources
    }
```

#### 2.4 输入部分

##### 2.4.1 Query Transform 

- Query改写

  可能是觉得用户提的问题不够具体，导致搜索的时候找到对应的信息；亦或者是用户的问题太复杂了，包含了很多层级的问题；亦或者是用户的问题太细致了，以至匹配不到相关的信息，需要对信息step back；

  ```python
  ############ 1.Query Rewriting 
  
  # Define the system prompt to guide the AI assistant's behavior
  system_prompt = "You are an AI assistant specialized in improving search queries. Your task is to rewrite user queries to be more specific, detailed, and likely to retrieve relevant information."
  
  # Define the user prompt with the original query to be rewritten
  user_prompt = f"""
  Rewrite the following query to make it more specific and detailed. Include relevant terms and concepts that might help in retrieving accurate information.
  
  Original query: {original_query}
  
  Rewritten query:
  """
  
  ############ 2. Step-back Prompting
  
  # Define the system prompt to guide the AI assistant's behavior
  system_prompt = "You are an AI assistant specialized in search strategies. Your task is to generate broader, more general versions of specific queries to retrieve relevant background information."
  
  # Define the user prompt with the original query to be generalized
  user_prompt = f"""
  Generate a broader, more general version of the following query that could help retrieve useful background information.
  
  Original query: {original_query}
  
  Step-back query:
  """
  
  ############ 3. Sub-query Decomposition
  # Define the system prompt to guide the AI assistant's behavior
  system_prompt = "You are an AI assistant specialized in breaking down complex questions. Your task is to decompose complex queries into simpler sub-questions that, when answered together, address the original query."
  
  # Define the user prompt with the original query to be decomposed
  user_prompt = f"""
  Break down the following complex query into {num_subqueries} simpler sub-queries. Each sub-query should focus on a different aspect of the original question.
  
  Original query: {original_query}
  
  Generate {num_subqueries} sub-queries, one per line, in this format:
  1. [First sub-query]
  2. [Second sub-query]
  And so on...
  """
  ```

  

##### 2.4.2 HyDE (Hypothetical Document Embedding)

 本质上还是Query改写的一种，具体操作方法如下：

- 根据用户的问题，假想一个document，这个document可以解决用户的query； (这里使用的还是LLM)
- 然后使用这个document进行向量化和相似度的比较，继续后面的流程；

```python
# Define the system prompt to instruct the model on how to generate the document
system_prompt = f"""You are an expert document creator. 
Given a question, generate a detailed document that would directly answer this question.
The document should be approximately {desired_length} characters long and provide an in-depth, 
informative answer to the question. Write as if this document is from an authoritative source
on the subject. Include specific details, facts, and explanations.
Do not mention that this is a hypothetical document - just write the content directly."""

# Define the user prompt with the query
user_prompt = f"Question: {query}\n\nGenerate a document that fully answers this question:"
```
