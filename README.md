# Multi-RAG과 Multi-Region LLM으로 한국어 Chatbot 만들기 

[Amazon Bedrock](https://aws.amazon.com/ko/bedrock/)의 Anthropic Claude LLM(Large Language Models) 모델을 이용하여 질문/답변(Question/Answering)을 수행하는 Chatbot을 [Knowledge Database](https://aws.amazon.com/ko/about-aws/whats-new/2023/09/knowledge-base-amazon-bedrock-models-data-sources/)를 이용하여 구현합니다. 대량의 데이터로 사전학습(pretrained)한 대규모 언어 모델(LLM)은 학습되지 않은 질문에 대해서도 가장 가까운 답변을 맥락(context)에 맞게 찾아 답변할 수 있습니다. 또한, [RAG(Retrieval-Augmented Generation)](https://docs.aws.amazon.com/ko_kr/sagemaker/latest/dg/jumpstart-foundation-models-customize-rag.html)를 이용하면 LLM은 잘못된 답변(hallucination)의 영향을 줄일 수 있으며, [파인 튜닝(fine tuning)](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-fine-tuning.html)을 제공하는 것처럼 최신의 데이터를 활용할 수 있습니다. RAG는 [prompt engineering](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-customize-prompt-engineering.html) 기술 중의 하나로서 vector store를 지식 데이터베이스로 이용하고 있습니다. 

Vector store는 이미지, 문서(text document), 오디오와 같은 구조화 되지 않은 컨텐츠(unstructured content)를 저장하고 검색할 수 있습니다. 특히 대규모 언어 모델(LLM)의 경우에 embedding을 이용하여 텍스트들의 연관성(sementic meaning)을 벡터(vector)로 표현할 수 있으므로, 연관성 검색(sementic search)을 통해 질문에 가장 가까운 답변을 찾을 수 있습니다. 여기서는 대표적인 In-memory vector store인 [Faiss](https://github.com/facebookresearch/faiss/wiki/Getting-started)와 persistent store이면서 대용량 병렬처리가 가능한 [Amazon OpenSearch](https://medium.com/@pandey.vikesh/rag-ing-success-guide-to-choose-the-right-components-for-your-rag-solution-on-aws-223b9d4c7280)와 완전관리형 검색서비스인 Kendra를 이용하여 RAG를 구현합니다.

Multiple RAG에서는 각 RAG가 관련 문서(Relevant Documents)를 생성하게 되므로, 전체 관련 문서들에서 실제 LLM에 전달할 문서의 숫자를 제한하여야 합니다. 이것은 Cluade2.1에서 200k token을 제공함에도 여전히 LLM의 context window에는 제한이 있기 때문입니다. 2023년 re:Invent에서는 다양한 Aurora, OpenSearch, Kendra뿐 아니라 Document DB, Mongo DB, Neptune등 거의 모든 데이터베이스의 RAG 지원이 발표되었습니다. 따라서, 향후 다양한 Database가 RAG의 소스로서 제공될 수 있습니다. 

Multiple LLM을 사용하게 되는 케이스에는 1) 다른 종류의 LLM을 사용하는 케이스 2) 같은 LLM을 다른 리전에서 사용하는 케이스가 있을 수 있습니다. LLM마다 잘하는 범위가 다를수 있으므로 다른 LLM의 결과를 통합해야할 수 있습니다. 또한, Bedrock On Demend의 경우에 단위시간당 보낼수 있는 Reuqest의 숫자 및 Token의 숫자가 제한되므로, 다른 리전의 동일 모델을 사용할 수 있다면, Provisioned로 사용하기 어려운 경우에 유용하게 이용될 수 있습니다. 1)의 경우에는 동시에 같은 요청을 보내서 다른 응답을 통합하는 과정이 필요하며, 2의 경우에는 Load를 분배하고 관리하는 방법이 필요합니다.

실행시간을 단축하기 위하여 Multi Thread를 사용합니다. [Lambda의 Multi thread](https://aws.amazon.com/ko/blogs/compute/parallel-processing-in-python-with-aws-lambda/)을 사용하므로써 연속적인 작업(sequencial job)을 병렬처리 할 수 있습니다.

[Lambda-Parallel Processing](https://aws.amazon.com/ko/blogs/compute/parallel-processing-in-python-with-aws-lambda/)와 같이 Lambda에서는 Pipe()을 이용합니다. 

## 아키텍처 개요

전체적인 아키텍처는 아래와 같습니다. 사용자는 Amazon CloudFront를 통해 [Amazon S3](https://aws.amazon.com/ko/s3/)로 부터 웹페이지에 필요한 리소스를 읽어옵니다. 사용자가 Chatbot 웹페이지에 로그인을 하면, 사용자 아이디를 이용하여 Amazon DynamoDB에 저장된 채팅 이력을 로드합니다. 이후 사용자가 메시지를 입력하면 WebSocket을 이용하여 LLM에 질의를 하게 되는데, DynamoDB로 부터 읽어온 채팅 이력과 RAG를 제공하는 Vector Database로부터 읽어온 관련문서(Relevant docs)를 이용하여 적절한 응답을 얻습니다. 이러한 RAG 구현을 위하여 [LangChain을 활용](https://python.langchain.com/docs/get_started/introduction.html)하여 Application을 개발하였고, Chatbot을 제공하는 인프라는 [AWS CDK](https://aws.amazon.com/ko/cdk/)를 통해 배포합니다. 

<img src="./images/basic-architecture.png" width="800">

상세하게 단계별로 설명하면 아래와 같습니다.

단계1: 사용자가 채팅창에서 질문을 입력합니다. 

단계2: [lambda(chat)](./lambda-chat-ws/lambda_function.py)은 기존 채팅이력을 DynamoDB에서 읽어오고, Assistant와의 상호작용(interaction)을 고려한 새로운 질문을 생성합니다.

단계3: RAG를 위한 지식저장소(Knowledge Store)인 Faiss, Amazon Kendra, Amazon OpenSearch로 부터 관련된 문서(Relevant Documents)를 조회합니다.

단계4: Bedrock의 Claude LLM으로 질문을 전달합니다. 여기서는 On Demend 방식의 용량증대를 위하여 각 event는 us-east-1과 us-west-2의 Bedrock으로 번갈아서 요청을 보내게 됩니다. 

단계5: lambda(chat)은 LLM의 응답을 Diaglog로 저장하고, 사용자에게 답변을 전달합니다.

단계6: 사용자는 답변을 확인하고, 필요시 Amazon S3에 저장된 문서를 Amazon CloudFront를 이용해 안전하게 읽어서 보여줍니다. 


이때의 Sequence diagram은 아래와 같습니다.

<img src="./images/sequence" width="800">

## 주요 구성

### Multi Region LLM을 LangChain으로 연결

LangChain의 [Bedrock](https://python.langchain.com/docs/integrations/providers/bedrock)을 import하여 LLM과 application을 연결합니다. 여기서는 LLM Endpoint의 용량을 늘릴수 있도록 여러 Region의 LLM을 사용하고자 하며, 예제에서는 "us-east-1"과, "us-west-2"를 아래와 같이 정의하여 사용합니다. 

[cdk-multi-rag-chatbot-stack.ts](./cdk-multi-rag-chatbot/lib/cdk-multi-rag-chatbot-stack.ts)에서는 아래와 같이 LLM의 profile을 저장한 후에 LLM을 처리하는 [lambda(chat)](./lambda-chat-ws/lambda_function.py)에 관련 정보를 Environment variables로 전달합니다. 

```typescript
const profile_of_LLMs = JSON.stringify([
  {
    "bedrock_region": "us-west-2",
    "model_type": "claude",
    "model_id": "anthropic.claude-v2:1",
    "maxOutputTokens": "8196"
  },
  {
    "bedrock_region": "us-east-1",
    "model_type": "claude",
    "model_id": "anthropic.claude-v2:1",
    "maxOutputTokens": "8196"
  },
]);
```

사용자가 보낸 메시지가 lambda(chat)에 event로 전달되면 아래와 같이 bedrock client를 정의한 후에, LangChain으로 Bedrock과 BedrockEmbeddings를 정의합니다. 

```python
profile_of_LLMs = json.loads(os.environ.get('profile_of_LLMs'))
selected_LLM = 0

profile = profile_of_LLMs[selected_LLM]
bedrock_region = profile['bedrock_region']
modelId = profile['model_id']

boto3_bedrock = boto3.client(
    service_name = 'bedrock-runtime',
    region_name = bedrock_region,
    config = Config(
        retries = {
            'max_attempts': 30
        }
    )
)
parameters = get_parameter(profile['model_type'], int(profile['maxOutputTokens']))

llm = Bedrock(
    model_id = modelId,
    client = boto3_bedrock,
    streaming = True,
    callbacks = [StreamingStdOutCallbackHandler()],
    model_kwargs = parameters)

bedrock_embeddings = BedrockEmbeddings(
    client = boto3_bedrock,
    region_name = bedrock_region,
    model_id = 'amazon.titan-embed-text-v1'
)      
```

Lambda(chat)은 event를 받을때마다 아래와 같이 새로운 LLM으로 교차하게되므로, 하나의 region에서 LLM을 처리할때보다 용량을 N개의 region에서 LLM을 사용하게 되면 N배의 용량이 증가하게 됩니다. 

```python
if selected_LLM >= number_of_LLMs - 1:
    selected_LLM = 0
else:
    selected_LLM = selected_LLM + 1
```


## RAG를 위한 Knowledge Store의 정의

여기서는 Knowledge Store로 OpenSearch, Faiss, Kendra를 활용합니다. Knowledge Store는 application에 맞게 추가하거나 제외할 수 있습니다.

### OpenSearch

[OpenSearchVectorSearch](https://api.python.langchain.com/en/latest/vectorstores/langchain.vectorstores.opensearch_vector_search.OpenSearchVectorSearch.html)을 이용해 vector store를 정의합니다. 

```python
from langchain.vectorstores import OpenSearchVectorSearch

vectorstore_opensearch = OpenSearchVectorSearch(
    index_name = "rag-index-*", # all
    is_aoss = False,
    ef_search = 1024, # 512(default )
    m = 48,
    #engine = "faiss",  # default: nmslib
    embedding_function = bedrock_embeddings,
    opensearch_url = opensearch_url,
    http_auth = (opensearch_account, opensearch_passwd), # http_auth = awsauth,
)
```

OpenSearch를 이용한 vector store에 데이터는 아래와 같이 add_documents()로 넣을 수 있습니다. index에 userId를 넣으면, 검색할때에 특정 사용자가 올린 문서만을 참조할 수 있습니다. 

```python
def store_document_for_opensearch(bedrock_embeddings, docs, userId, requestId):
    new_vectorstore = OpenSearchVectorSearch(
        index_name="rag-index-"+userId+'-'+requestId,
        is_aoss = False,
        #engine="faiss",  # default: nmslib
        embedding_function = bedrock_embeddings,
        opensearch_url = opensearch_url,
        http_auth=(opensearch_account, opensearch_passwd),
    )
    new_vectorstore.add_documents(docs)    
```

관련된 문서(relevant docs)는 아래처럼 검색할 수 있습니다. 문서가 검색이 되면 아래와 같이 metadata에서 문서의 이름(title), 페이지(_excerpt_page_number), 파일의 경로(source) 및 발췌문(excerpt)를 추출해서 관련된 문서(Relevant Document)에 추가할 수 있습니다.

```python
relevant_documents = vectorstore_opensearch.similarity_search_with_score(
    query = query,
    k = top_k,
)

for i, document in enumerate(relevant_documents):
    name = document[0].metadata['name']
    page = document[0].metadata['page']
    uri = document[0].metadata['uri']

    excerpt = document[0].page_content
    confidence = str(document[1])
    assessed_score = str(document[1])

    doc_info = {
        "rag_type": rag_type,
        "confidence": confidence,
        "metadata": {
            "source": uri,
            "title": name,
            "excerpt": excerpt,
            "document_attributes": {
                "_excerpt_page_number": page
            }
        },
        "assessed_score": assessed_score,
    }
    relevant_docs.append(doc_info)

return relevant_docs
```

이때, assessed_score의 한 예는 "0.008877229"와 같은 소숫점 이하의 숫자를 가집니다.

### Faiss

아래와 같이 Faiss에 문서를 처음 등록할 때에 vector store로 정의합니다. 이후로 추가되는 문서는 아래처럼 add_documents를 이용해 추가합니다. Faiss는 in-memory vectore store로 인스턴스가 유지될 동안만 사용할 수 있습니다. 

```python
if isReady == False:
    embeddings = bedrock_embeddings
    vectorstore_faiss = FAISS.from_documents( # create vectorstore from a document
        docs,  # documents
        embeddings  # embeddings
    )
    isReady = True
else:
    vectorstore_faiss.add_documents(docs)    
```

similarity_search_with_score()를 이용면 similarity에 대한 score를 얻을 수 있습니다. Faiss는 관련도가 높은 순서로 문서를 전달하는데, 관련도가 높을 수도록 score의 값는 작은값을 가집니다. 문서에서 이름(title), 페이지(_excerpt_page_number), 신뢰도(assessed_score), 발췌문(excerpt)을 추출합니다. 

```python
relevant_documents = vectorstore_faiss.similarity_search_with_score(
    query = query,
    k = top_k,
)

for i, document in enumerate(relevant_documents):    
    name = document[0].metadata['name']
    page = document[0].metadata['page']
    uri = document[0].metadata['uri']
    confidence = int(document[1])
    assessed_score = int(document[1])

    doc_info = {
        "rag_type": rag_type,
        "confidence": confidence,
        "metadata": {
            "source": uri,
            "title": name,
            "excerpt": document[0].page_content,
            "document_attributes": {
                "_excerpt_page_number": page
            }
        },
        "assessed_score": assessed_score,
    }
```

Faiss의 assessed_score는 56와 같은 1보다 큰 정수값을 가집니다.

### Kendra

Kendra에 문서를 넣을때는 아래와 같이 S3 bucekt를 이용합니다. 경로를 생성할때에 파일명은 URL encoding을 하여야 합니다. 소스으 경로(source_uri)은 CloudFront와 연결된 S3의 경로를 이용합니다. Kendra에 저장되는 문서는 알애ㅘ 같은 파일포맷으로 변환하여야 합니다. 파일을 올릴때는 [boto3의 batch_put_document()](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/kendra/client/batch_put_document.html)을 이용합니다. 


```python
def store_document_for_kendra(path, s3_file_name, requestId):
    encoded_name = parse.quote(s3_file_name)
    source_uri = path + encoded_name    
    ext = (s3_file_name[s3_file_name.rfind('.')+1:len(s3_file_name)]).upper()

    # PLAIN_TEXT, XSLT, MS_WORD, RTF, CSV, JSON, HTML, PDF, PPT, MD, XML, MS_EXCEL
    if(ext == 'PPTX'):
        file_type = 'PPT'
    elif(ext == 'TXT'):
        file_type = 'PLAIN_TEXT'         
    elif(ext == 'XLS' or ext == 'XLSX'):
        file_type = 'MS_EXCEL'      
    elif(ext == 'DOC' or ext == 'DOCX'):
        file_type = 'MS_WORD'
    else:
        file_type = ext

    kendra_client = boto3.client(
        service_name='kendra', 
        region_name=kendra_region,
        config = Config(
            retries=dict(
                max_attempts=10
            )
        )
    )

    documents = [
        {
            "Id": requestId,
            "Title": s3_file_name,
            "S3Path": {
                "Bucket": s3_bucket,
                "Key": s3_prefix+'/'+s3_file_name
            },
            "Attributes": [
                {
                    "Key": '_source_uri',
                    'Value': {
                        'StringValue': source_uri
                    }
                },
                {
                    "Key": '_language_code',
                    'Value': {
                        'StringValue': "ko"
                    }
                },
            ],
            "ContentType": file_type
        }
    ]

    result = kendra_client.batch_put_document(
        IndexId = kendraIndex,
        RoleArn = roleArn,
        Documents = documents       
    )
```    

Langchain의 [Kendra Retriever](https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.kendra.AmazonKendraRetriever.html)로 아래와 같이 Kendra Retriever를 생성합니다. 파일을 등록할때와 동일하게 "_language_code"을 "ko"로 설정하고, "top_k"만큼의 Relevent Document를 가져오도록 설정합니다.

```python
from langchain.retrievers import AmazonKendraRetriever
kendraRetriever = AmazonKendraRetriever(
    index_id=kendraIndex, 
    top_k=top_k, 
    region_name=kendra_region,
    attribute_filter = {
        "EqualsTo": {      
            "Key": "_language_code",
            "Value": {
                "StringValue": "ko"
            }
        },
    },
)
```

get_relevant_documents()을 이용하여 관련된 문서를 가져옵니다. LangChain의 AmazonKendraRetriever은 Kendra의 Retrieve를 이용해 조회를 합니다. 


```python
rag_type = "kendra"
api_type = "kendraRetriever"
relevant_docs = []
relevant_documents = kendraRetriever.get_relevant_documents(
    query = query,
    top_k = top_k,
)

for i, document in enumerate(relevant_documents):
    #print('document.page_content:', document.page_content)

    result_id = document.metadata['result_id']
    document_id = document.metadata['document_id']
    title = document.metadata['title']
    excerpt = document.metadata['excerpt']
    uri = document.metadata['document_attributes']['_source_uri']
    page = document.metadata['document_attributes']['_excerpt_page_number']

    assessed_score = ""

    doc_info = {
        "rag_type": rag_type,
        "api_type": api_type,
        "metadata": {
            "document_id": document_id,
            "source": uri,
            "title": title,
            "excerpt": excerpt,
            "document_attributes": {
                "_excerpt_page_number": page
            }
        },
        "assessed_score": assessed_score,
        "result_id": result_id
    }
    relevant_docs.append(doc_info)

return relevant_docs
```


### 관련된 문서를 포함한 RAG 구현

Assistent와 상호작용(interacton)을 위하여 채팅이력을 이용해 사용자의 질문을 새로운 질문(revised_question)으로 업데이트합니다. 이때 사용하는 prompt는 한국어와 영어로 나누어 아래처럼 적용하고 있습니다. 

```python
revised_question = get_revised_question(llm, connectionId, requestId, text) # generate new prompt 

def get_revised_question(llm, connectionId, requestId, query):    
    pattern_hangul = re.compile('[\u3131-\u3163\uac00-\ud7a3]+')
    word_kor = pattern_hangul.search(str(query))

    if word_kor and word_kor != 'None':
        condense_template = """
        <history>
        {chat_history}
        </history>

        Human: <history>를 참조하여, 다음의 <question>의 뜻을 명확히 하는 새로운 질문을 한국어로 생성하세요.

        <question>            
        {question}
        </question>
            
        Assistant: 새로운 질문:"""
    else: 
        condense_template = """
        <history>
        {chat_history}
        </history>
        Answer only with the new question.

        Human: using <history>, rephrase the follow up <question> to be a standalone question.
         
        <quesion>
        {question}
        </question>

        Assistant: Standalone question:"""

    condense_prompt_claude = PromptTemplate.from_template(condense_template)        
    condense_prompt_chain = LLMChain(llm=llm, prompt=condense_prompt_claude)

    chat_history = extract_chat_history_from_memory()
    revised_question = condense_prompt_chain.run({"chat_history": chat_history, "question": query})
    
    return revised_question
```

관련된 문서를 아래와 같이 Kendra와 Vector Store인 Faiss, OpenSearch에서 top_k개 만큼 가져옵니다. 

```python
def retrieve_process_from_RAG(conn, query, top_k, rag_type):
    relevant_docs = []
    if rag_type == 'kendra':
        rel_docs = retrieve_from_kendra(query=query, top_k=top_k)      
        print('rel_docs (kendra): '+json.dumps(rel_docs))
    else:
        rel_docs = retrieve_from_vectorstore(query=query, top_k=top_k, rag_type=rag_type)
        print(f'rel_docs ({rag_type}): '+json.dumps(rel_docs))
    
    if(len(rel_docs)>=1):
        for doc in rel_docs:
            relevant_docs.append(doc)    
    
    conn.send(relevant_docs)
    conn.close()
```

검색을 병렬화하면 속도를 개선할 수 있습니다. 아래와 multiprocessing을 이용해 여러개의 thread를 생성하여 검색하고 결과는 Pipe를 이용하여 얻습니다.

```python
from multiprocessing import Process, Pipe

processes = []
parent_connections = []
for rag in capabilities:
    parent_conn, child_conn = Pipe()
parent_connections.append(parent_conn)

process = Process(target = retrieve_process_from_RAG, args = (child_conn, revised_question, top_k, rag))
processes.append(process)

for process in processes:
    process.start()

for parent_conn in parent_connections:
    rel_docs = parent_conn.recv()

if (len(rel_docs) >= 1):
    for doc in rel_docs:
        relevant_docs.append(doc)

for process in processes:
    process.join()
```


여기서는 3개의 Vector Store를 사용하고 top_k씩 가져오므로 최대 3xtop_k의 문서를 얻게 됩니다. 문서가 너무 많으면 context window의 범위를 넘을 수 있으므로 여기서 가장 관련이 있는 top_k만을 아래와 같이 선정합니다. 아래에서는 Faiss의 similarity search로 top_k의 문서중에 score가 200이하 인것만을 관련된 문서로 선택하여 사용하고 있습니다.

```python
if len(relevant_docs) >= 1:
    relevant_docs = check_confidence(revised_question, relevant_docs, bedrock_embeddings)

def check_confidence(query, relevant_docs, bedrock_embeddings):
    excerpts = []
    for i, doc in enumerate(relevant_docs):
        excerpts.append(
            Document(
                page_content=doc['metadata']['excerpt'],
                metadata={
                    'name': doc['metadata']['title'],
                    'order':i,
                }
            )
        )  

    embeddings = bedrock_embeddings
    vectorstore_confidence = FAISS.from_documents(
        excerpts,  # documents
        embeddings  # embeddings
    )            
    rel_documents = vectorstore_confidence.similarity_search_with_score(
        query=query,
        k=top_k
    )

    docs = []
    for i, document in enumerate(rel_documents):
        order = document[0].metadata['order']
        name = document[0].metadata['name']
        assessed_score = document[1]

        relevant_docs[order]['assessed_score'] = int(assessed_score)

        if assessed_score < 200:
            docs.append(relevant_docs[order])    

    return docs
```

관련된 문서들은 아래와 같이 하나의 context로 모읍니다. 이후 PROMPT에 relevant_context와 새로운 질문(revised_question)을 넣어서 LLM에 답변을 요청합니다. 답변은 stream으로 받아서 아래처럼 client로 전달합니다.

```python
relevant_context = ""
for document in relevant_docs:
    relevant_context = relevant_context + document['metadata']['excerpt'] + "\n\n"

stream = llm(PROMPT.format(context=relevant_context, question=revised_question))
msg = readStreamMsg(connectionId, requestId, stream)
```




### Reference 표시하기

아래와 같이 kendra는 doc의 metadata에서 reference에 대한 정보를 추출합니다. 여기서 file의 이름은 doc.metadata['title']을 이용하고, 페이지는 doc.metadata['document_attributes']['_excerpt_page_number']을 이용해 얻습니다. URL은 cloudfront의 url과 S3 bucket의 key, object를 조합하여 구성합니다. opensearch와 faiss는 파일명, page 숫자, 경로(URL path)를 metadata의 'name', 'page', 'url'을 통해 얻습니다.

```python
def get_reference(docs):
    reference = "\n\nFrom\n"
    for i, doc in enumerate(docs):
        name = doc['metadata']['title']
        uri = doc['metadata']['source']
        reference = reference + f"{i+1}. <a href={uri} target=_blank>{name} </a>, {doc['rag_type']} ({doc['assessed_score']})\n"

return reference
```


### AWS CDK로 인프라 구현하기

[CDK 구현 코드](./cdk-qa-with-rag/README.md)에서는 Typescript로 인프라를 정의하는 방법에 대해 상세히 설명하고 있습니다.

## 직접 실습 해보기

### 사전 준비 사항

이 솔루션을 사용하기 위해서는 사전에 아래와 같은 준비가 되어야 합니다.

- [AWS Account 생성](https://repost.aws/ko/knowledge-center/create-and-activate-aws-account)


### CDK를 이용한 인프라 설치
[인프라 설치](https://github.com/kyopark2014/question-answering-chatbot-using-RAG-based-on-LLM/blob/main/deployment.md)에 따라 CDK로 인프라 설치를 진행합니다. 


### 실행결과

[fsi_faq_ko.csv](https://github.com/kyopark2014/korean-chatbot-using-amazon-bedrock/blob/main/fsi_faq_ko.csv)을 다운로드한 후에 파일 아이콘을 선택하여 업로드하면 Knowledge Database에 저장됩니다. 이후 아래와 같이 파일 내용을 확인할 수 있도록 요약하여 보여줍니다.

![image](https://github.com/kyopark2014/korean-chatbot-using-amazon-bedrock/assets/52392004/a0b3b5b8-6e1e-4240-9ee4-e539680fa28d)

채팅창에 "간편조회 서비스를 영문으로 사용할 수 있나요?” 라고 입력합니다. 이때의 결과는 ＂아니오”입니다. 이때의 결과는 아래와 같습니다.

![image](https://github.com/kyopark2014/korean-chatbot-using-amazon-bedrock/assets/52392004/c7aeca05-0209-49c3-9df9-7e04026900f2)

채팅창에 "이체를 할수 없다고 나옵니다. 어떻게 해야 하나요?” 라고 입력하고 결과를 확인합니다.

![image](https://github.com/kyopark2014/korean-chatbot-using-amazon-bedrock/assets/52392004/56ad9192-6b7c-49c7-9289-b6a3685cb7d4)

채팅창에 "공동인증서 창구발급 서비스는 무엇인가요?” 라고 입력하면 아래와 같은 결과를 얻을 수 있습니다.

![image](https://github.com/kyopark2014/korean-chatbot-using-amazon-bedrock/assets/52392004/95a78e6a-5a78-4879-98a1-a30aa6f7e3d5)



아래와 같이 채팅창에 "이체를 할수 없다고 나옵니다. 어떻게 해야 하나요?” 라고 입력하고 결과를 확인합니다.

![image](https://github.com/kyopark2014/rag-chatbot-using-bedrock-claude-and-kendra/assets/52392004/849169c3-c9ee-40dd-8e44-0ec308688ce6)


채팅창에 "간편조회 서비스를 영문으로 사용할 수 있나요?” 라고 입력합니다. "영문뱅킹에서는 간편조회서비스 이용불가"하므로 좀더 자세한 설명을 얻었습니다.

![image](https://github.com/kyopark2014/rag-chatbot-using-bedrock-claude-and-kendra/assets/52392004/3a896488-af0c-42b2-811b-d2c0debf5462)

채팅창에 "공동인증서 창구발급 서비스는 무엇인가요?"라고 입력하고 결과를 확인합니다.

![image](https://github.com/kyopark2014/rag-chatbot-using-bedrock-claude-and-kendra/assets/52392004/2e2b2ae1-7c50-4c14-968a-6c58332d99af)





## 리소스 정리하기 

더이상 인프라를 사용하지 않는 경우에 아래처럼 모든 리소스를 삭제할 수 있습니다. 

1) [API Gateway Console](https://ap-northeast-2.console.aws.amazon.com/apigateway/main/apis?region=ap-northeast-2)로 접속하여 "api-chatbot-for-multi-rag-chatbot", "api-multi-rag-chatbot"을 삭제합니다.

2) [Cloud9 console](https://ap-northeast-2.console.aws.amazon.com/cloud9control/home?region=ap-northeast-2#/)에 접속하여 아래의 명령어로 전체 삭제를 합니다.


```text
cd ~/environment/multi-rag-and-multi-region-llm-for-chatbot/cdk-multi-rag-chatbot/ && cdk destroy --all
```


## 결론


## 실습 코드 및 도움이 되는 참조 블로그

아래의 링크에서 실습 소스 파일 및 기계 학습(ML)과 관련된 자료를 확인하실 수 있습니다.

- [Amazon SageMaker JumpStart를 이용하여 Falcon Foundation Model기반의 Chatbot 만들기](https://aws.amazon.com/ko/blogs/tech/chatbot-based-on-falcon-fm/)
- [Amazon SageMaker JumpStart와 Vector Store를 이용하여 Llama 2로 Chatbot 만들기](https://aws.amazon.com/ko/blogs/tech/sagemaker-jumpstart-vector-store-llama2-chatbot/)
- [VARCO LLM과 Amazon OpenSearch를 이용하여 한국어 Chatbot 만들기](https://aws.amazon.com/ko/blogs/tech/korean-chatbot-using-varco-llm-and-opensearch/)
- [Amazon Bedrock을 이용하여 Stream 방식의 한국어 Chatbot 구현하기](https://aws.amazon.com/ko/blogs/tech/stream-chatbot-for-amazon-bedrock/)



## Reference 

[Claude - Constructing a prompt](https://docs.anthropic.com/claude/docs/constructing-a-prompt)
