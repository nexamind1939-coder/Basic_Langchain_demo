# Basic_Langchain_demo
import os
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# 1. Setup
os.environ["OPENAI_API_KEY"] = "your-openai-api-key-here"
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 2. Updated Prompt Template with a History Placeholder
# We must use MessagesPlaceholder so LangChain knows exactly where to inject the past chat history.
prompt_template = ChatPromptTemplate.from_messages([
    ("system", "You are an expert chef. Help the user with recipes and cooking advice based on their conversation history."),
    MessagesPlaceholder(variable_name="chat_history"),
    ("user", "{input}")
])

# 3. Build the Base Chain
base_chain = prompt_template | llm | StrOutputParser()

# 4. Manage Memory / Chat History Storage
# In a real app, this would be a database. For this basic app, we use a dictionary in memory.
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# 5. Wrap the Chain with Message History
# This automatically feeds past messages into 'chat_history' and saves new ones.
conversational_chain = RunnableWithMessageHistory(
    base_chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)

# 6. Create an Interactive Terminal Chat Loop
print("--- AI Chef Ready! (Type 'exit' or 'quit' to stop) ---\n")

# We give our conversation a unique session ID
config = {"configurable": {"session_id": "user_cooking_session"}}

while True:
    user_input = input("You: ")
    if user_input.lower() in ["exit", "quit"]:
        print("Chef: Happy cooking! Goodbye!")
        break
        
    print("Chef: ", end="", flush=True)
    
    # We call .stream() on the conversational chain
    for chunk in conversational_chain.stream({"input": user_input}, config=config):
        print(chunk, end="", flush=True)
    print("\n") # New line after streaming finishes
    
import os
import streamlit as st
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# --- 1. Page Configuration & Setup ---
st.set_page_config(page_title="AI Chef Assistant", page_icon="🍳", layout="centered")
st.title("🍳 Your AI Chef Assistant")
st.caption("Ask for recipes, cooking tips, or modification ideas based on your history!")

# Set your API key
os.environ["OPENAI_API_KEY"] = "your-openai-api-key-here"

# --- 2. Initialize LangChain Components ---
# We use st.cache_resource so Streamlit doesn't recreate the model object on every click
@st.cache_resource
def init_langchain():
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
    
    prompt_template = ChatPromptTemplate.from_messages([
        ("system", "You are an expert chef. Help the user with recipes and cooking advice based on their conversation history."),
        MessagesPlaceholder(variable_name="chat_history"),
        ("user", "{input}")
    ])
    
    base_chain = prompt_template | llm | StrOutputParser()
    return base_chain

base_chain = init_langchain()

# --- 3. Manage Memory using Streamlit Session State ---
# Streamlit clears variables when the page reruns, so we store history in st.session_state
if "chat_history" not in st.session_state:
    st.session_state.chat_history = InMemoryChatMessageHistory()

def get_session_history(session_id: str):
    return st.session_state.chat_history

# Wrap our chain with history
conversational_chain = RunnableWithMessageHistory(
    base_chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)

# --- 4. Render Chat History on UI ---
# Show all previous messages stored in session state
for msg in st.session_state.chat_history.messages:
    # LangChain messages have a 'type' property (human or ai)
    role = "user" if msg.type == "human" else "assistant"
    with st.chat_message(role):
        st.markdown(msg.content)

# --- 5. Handle New User Input ---
if user_input := st.chat_input("What ingredients do you have?"):
    
    # Display user's question instantly
    with st.chat_message("user"):
        st.markdown(user_input)
        
    # Generate and stream the AI assistant's response
    with st.chat_message("assistant"):
        config = {"configurable": {"session_id": "streamlit_session"}}
        
        # st.write_stream consumes a python generator (like .stream()) and displays it live
        response = st.write_stream(
            conversational_chain.stream({"input": user_input}, config=config)
        )
