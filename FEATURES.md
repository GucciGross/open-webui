
# Open WebUI Features

This document provides an overview of the features available in Open WebUI, how to implement them in other applications, and how to extend the codebase with new functionality.

## Core Features

Open WebUI is a powerful and extensible web interface for language models. It provides a wide range of features that can be customized and integrated into other applications.

### 1. User Management

Open WebUI includes a comprehensive user management system that allows administrators to manage users, roles, and permissions.

**Key Files:**

*   **Frontend:**
    *   `src/routes/(app)/admin/users/+page.svelte`: The main user management interface.
    *   `src/lib/components/admin/Users/UserList.svelte`: Component for listing and managing users.
    *   `src/lib/components/admin/Users/UserList/EditUserModal.svelte`: Modal for editing user details, including their role.
    *   `src/lib/components/admin/Users/Groups.svelte`: Component for managing user groups and permissions.
*   **Backend:**
    *   `backend/open_webui/routers/users.py`: API endpoints for user management.
    *   `backend/open_webui/routers/groups.py`: API endpoints for group management.
    *   `backend/open_webui/models/users.py`: Database model for users.
    *   `backend/open_webui/models/groups.py`: Database model for groups.

**Implementation in Other Apps:**

To implement a similar user management system in your own application, you would need to:

1.  **Create a User Model:** Define a user model with fields for username, email, password, and role.

    ```python
    # backend/open_webui/models/users.py
    from pydantic import BaseModel
    from typing import Optional

    class User(BaseModel):
        id: str
        name: str
        email: str
        role: str
        profile_image_url: Optional[str] = None
    ```

2.  **Implement Authentication:** Use a library like `passlib` for password hashing and JWT for session management.

    ```python
    # backend/open_webui/utils/auth.py
    from passlib.context import CryptContext
    from jose import JWTError, jwt

    pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

    def verify_password(plain_password, hashed_password):
        return pwd_context.verify(plain_password, hashed_password)

    def get_password_hash(password):
        return pwd_context.hash(password)

    def create_access_token(data: dict):
        to_encode = data.copy()
        return jwt.encode(to_encode, "SECRET_KEY", algorithm="HS256")
    ```

3.  **Build API Endpoints:** Create API endpoints for creating, reading, updating, and deleting users.

    ```python
    # backend/open_webui/routers/users.py
    from fastapi import APIRouter, Depends
    from ..models.users import User
    from ..utils.auth import get_current_user

    router = APIRouter()

    @router.get("/", response_model=list[User])
    async def get_users(current_user: User = Depends(get_current_user)):
        # Your logic to get all users
        pass

    @router.put("/{user_id}", response_model=User)
    async def update_user(user_id: str, user: User, current_user: User = Depends(get_current_user)):
        # Your logic to update a user
        pass
    ```

4.  **Develop a UI:** Create a user interface for administrators to manage users and roles. The `EditUserModal.svelte` component is a good example of how to build a UI for changing user roles.

    ```svelte
    <!-- src/lib/components/admin/Users/UserList/EditUserModal.svelte -->
    <script lang="ts">
      import { updateUser } from '$lib/apis/users';
      import Modal from '$lib/components/common/Modal.svelte';
      import Select from '$lib/components/common/Selector.svelte';

      export let user;
      export let onUpdate;

      let role;

      $: if (user) {
        role = user.role;
      }

      const handleUpdateUser = async () => {
        const res = await updateUser(user.id, { role });
        if (res) {
          onUpdate(res);
        }
      };
    </script>

    <Modal title="Edit User">
      <div class="p-4">
        <Select label="Role" options={['user', 'admin']} bind:value={role} />
        <button on:click={handleUpdateUser}>Update</button>
      </div>
    </Modal>
    ```

### 2. Chat Interface

The chat interface is the core of Open WebUI. It provides a rich user experience with support for Markdown, code highlighting, and file attachments.

**Key Files:**

*   **Frontend:**
    *   `src/routes/(app)/c/[id]/+page.svelte`: The main chat page.
    *   `src/lib/components/chat/Chat.svelte`: The main chat component.
    *   `src/lib/components/chat/Messages/Markdown.svelte`: Component for rendering Markdown messages. This component uses the `marked` library to render Markdown, and it's configured in `src/lib/utils/marked/index.ts` to support features like KaTeX for math equations and syntax highlighting for code blocks.
    *   `src/lib/components/chat/MessageInput.svelte`: Component for user input, including file attachments.
*   **Backend:**
    *   `backend/open_webui/routers/chats.py`: API endpoints for chat functionality.
    *   `backend/open_webui/models/chats.py`: Database model for chats.
    *   `backend/open_webui/socket/main.py`: Handles the WebSocket connection for real-time chat.

**Implementation in Other Apps:**

To implement a similar chat interface, you would need to:

1.  **Use a WebSocket Library:** Use a library like `socket.io` to enable real-time communication between the client and server.

    ```python
    # backend/open_webui/socket/main.py
    import socketio

    sio = socketio.AsyncServer(async_mode="asgi", cors_allowed_origins="*")

    @sio.on("connect")
    async def connect(sid, environ):
        print("connect ", sid)

    @sio.on("message")
    async def message(sid, data):
        print("message ", data)
        await sio.emit("message", data, room=sid)

    @sio.on("disconnect")
    async def disconnect(sid):
        print("disconnect ", sid)
    ```

2.  **Render Markdown:** Use a library like `marked` to render Markdown messages. You can customize the renderer to support additional features like KaTeX and syntax highlighting.

    ```svelte
    <!-- src/lib/components/chat/Messages/Markdown.svelte -->
    <script lang="ts">
      import { marked } from '$lib/utils/marked';

      export let content;

      let renderedContent;

      $: {
        renderedContent = marked.parse(content);
      }
    </script>

    <div class="markdown-body">
      {@html renderedContent}
    </div>
    ```

3.  **Handle File Uploads:** Implement file upload functionality on both the client and server.

    ```python
    # backend/open_webui/routers/files.py
    from fastapi import APIRouter, UploadFile, File

    router = APIRouter()

    @router.post("/upload")
    async def upload_file(file: UploadFile = File(...)):
        # Your logic to save the file
        return {"filename": file.filename}
    ```

### 3. Document and Knowledge Base Management

Open WebUI allows users to upload and manage documents, creating a knowledge base that can be used to augment the language model's responses.

**Key Files:**

*   **Frontend:**
    *   `src/routes/(app)/workspace/knowledge/+page.svelte`: The main knowledge base management page.
    *   `src/lib/components/workspace/Knowledge/KnowledgeBase.svelte`: Component for managing knowledge bases.
*   **Backend:**
    *   `backend/open_webui/routers/knowledge.py`: API endpoints for knowledge base management.
    *   `backend/open_webui/retrieval`: Backend code for document retrieval and processing. This includes loaders for various file types and integrations with vector databases.

**Implementation in Other Apps:**

1.  **Document Parsing:** Use libraries like `pypdf` and `python-docx` to parse different document formats. The `backend/open_webui/retrieval/loaders` directory contains examples of how to load different document types.

    ```python
    # backend/open_webui/retrieval/loaders/main.py
    from langchain.document_loaders import PyPDFLoader, TextLoader, UnstructuredWordDocumentLoader

    def load_document(file_path: str):
        if file_path.endswith(".pdf"):
            loader = PyPDFLoader(file_path)
        elif file_path.endswith(".docx"):
            loader = UnstructuredWordDocumentLoader(file_path)
        else:
            loader = TextLoader(file_path)
        return loader.load()
    ```

2.  **Vector Database:** Use a vector database like ChromaDB or Pinecone to store and query document embeddings. The `backend/open_webui/retrieval/vector/dbs` directory shows how to integrate with different vector databases.

3.  **Retrieval Augmented Generation (RAG):** Implement a RAG pipeline to retrieve relevant documents and use them to augment the language model's prompts.

    ```python
    # backend/open_webui/retrieval/main.py
    from langchain.chains import RetrievalQA
    from langchain.chat_models import ChatOllama
    from langchain.vectorstores import Chroma

    def create_rag_pipeline(embedding_function, persist_directory):
        llm = ChatOllama(model="llama2")
        vectorstore = Chroma(persist_directory=persist_directory, embedding_function=embedding_function)
        qa = RetrievalQA.from_chain_type(llm=llm, chain_type="stuff", retriever=vectorstore.as_retriever())
        return qa
    ```

### 4. Model Management

Open WebUI allows you to connect to and manage multiple language models.

**Key Files:**

*   **Frontend:**
    *   `src/routes/(app)/workspace/models/+page.svelte`: The main model management page.
    *   `src/lib/components/workspace/Models.svelte`: Component for listing and managing models.
*   **Backend:**
    *   `backend/open_webui/routers/models.py`: API endpoints for model management.

### 5. Prompt Library

You can create, save, and reuse prompts in Open WebUI.

**Key Files:**

*   **Frontend:**
    *   `src/routes/(app)/workspace/prompts/+page.svelte`: The main prompt library page.
    *   `src/lib/components/workspace/Prompts.svelte`: Component for managing prompts.
*   **Backend:**
    *   `backend/open_webui/routers/prompts.py`: API endpoints for prompt management.

### 6. Internationalization (i18n)

Open WebUI supports multiple languages.

**Key Files:**

*   `src/lib/i18n/`: This directory contains the translation files for different languages.
*   `src/lib/i18n/index.ts`: This file initializes the i18n library (`svelte-i18n`).

**Implementation in Other Apps:**

To add internationalization to your own Svelte application, you can use the `svelte-i18n` library. You will need to create JSON files for each language you want to support and then use the `_` store to translate your strings.

### 7. Theming

Open WebUI supports custom themes.

**Key Files:**

*   `static/themes/`: This directory contains the CSS files for different themes.

**Implementation in Other Apps:**

You can implement theming in your own application by creating different CSS files for each theme and then providing a way for the user to switch between them.

## Extending the Codebase

Open WebUI is designed to be extensible. You can add new features and integrations by following the existing code patterns.

### Example: Adding Stripe for Paid Roles

To add a feature where users can pay for a premium role using Stripe, you would need to:

1.  **Add Stripe to the Backend:**
    *   Install the Stripe Python library: `pip install stripe`
    *   Create a new router in `backend/open_webui/routers` for handling Stripe webhooks.

        ```python
        # backend/open_webui/routers/stripe.py
        from fastapi import APIRouter, Request, Header
        import stripe

        router = APIRouter()

        @router.post("/webhook")
        async def stripe_webhook(request: Request, stripe_signature: str = Header(None)):
            payload = await request.body()
            try:
                event = stripe.Webhook.construct_event(
                    payload=payload, sig_header=stripe_signature, secret="whsec_..."
                )
            except ValueError as e:
                # Invalid payload
                return {"error": str(e)}, 400
            except stripe.error.SignatureVerificationError as e:
                # Invalid signature
                return {"error": str(e)}, 400

            # Handle the event
            if event['type'] == 'checkout.session.completed':
                session = event['data']['object']
                # Fulfill the purchase...
                print(f"Payment for {session['amount_total']} succeeded!")

            return {"status": "success"}
        ```

    *   Create API endpoints for creating Stripe checkout sessions and handling payment success events.
    *   Update the user's role in the database when a payment is successful.

2.  **Add a Pricing Page to the Frontend:**
    *   Create a new route in `src/routes` for the pricing page.
    *   Use the Stripe.js library to create a checkout button.

        ```svelte
        <!-- src/routes/pricing/+page.svelte -->
        <script lang="ts">
          import { onMount } from 'svelte';
          import { loadStripe } from '@stripe/stripe-js';

          let stripe;

          onMount(async () => {
            stripe = await loadStripe('pk_test_...');
          });

          const createCheckoutSession = async () => {
            const res = await fetch('/api/stripe/checkout-session', { method: 'POST' });
            const { sessionId } = await res.json();
            await stripe.redirectToCheckout({ sessionId });
          };
        </script>

        <button on:click={createCheckoutSession}>
          Subscribe
        </button>
        ```

    *   Make an API call to your backend to create a checkout session.
    *   Redirect the user to the Stripe checkout page.

3.  **Update the UI to Reflect the User's Role:**
    *   Modify the UI to show different features or a different theme based on the user's role.
    *   You can use the `user.role` property in your Svelte components to conditionally render elements.

This is a high-level overview of how you could extend the codebase. For more detailed instructions, you can refer to the Stripe documentation and the existing Open WebUI codebase.
