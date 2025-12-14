### 🔴 How to build ai agent with tools and RAG without any ai agent framework except laravel   in  laravel api( react vite for frontend)

### Overview
Building an AI agent with tools and Retrieval-Augmented Generation (RAG) in a Laravel API backend (using PHP) and React with Vite frontend requires implementing the logic manually without third-party AI agent frameworks. We'll use Laravel's core features, its HTTP facade for API calls to an LLM (e.g., OpenAI), and a vector database like Qdrant for RAG (run it locally or via cloud API). The agent will handle tool calling by parsing LLM responses and executing PHP functions in a loop. For RAG, we'll embed queries/documents using OpenAI's embeddings API and retrieve similar content from Qdrant to augment prompts.

Assumptions:
- You have Laravel 12.x installed.
- OpenAI API key in `.env` (`OPENAI_API_KEY`).
- Qdrant running (e.g., locally at `http://127.0.0.1:6333` or cloud endpoint; install via Docker: `docker run -p 6333:6333 qdrant/qdrant`).
- Environment variables for models (e.g., `EMBEDDING_MODEL=text-embedding-ada-002`, `GPT_MODEL=gpt-4o-mini`, `EMBEDDING_MODEL_DIMS=1536`).
- Basic tools: e.g., "get_current_weather" and "search_web" as examples.
- RAG uses a simple document store per "collection" (e.g., user-specific).

This is a from-scratch implementation using direct API calls.

### Step 1: Set Up Laravel Backend
1. Create a new Laravel project:
   ```
   composer create-project laravel/laravel ai-agent-app
   cd ai-agent-app
   ```

2. Install Laravel's HTTP client (built-in, no extra packages needed).

3. Set up database (e.g., MySQL/PostgreSQL) for chat history if needed:
   ```
   php artisan migrate
   ```

4. Create a service class for the AI agent and RAG logic:
   ```
   php artisan make:class Services/AIAgentService
   ```

   In `app/Services/AIAgentService.php`:
   ```php
   <?php

   namespace App\Services;

   use Illuminate\Support\Facades\Http;
   use Illuminate\Support\Str;

   class AIAgentService
   {
       private $openaiApiKey;
       private $qdrantEndpoint = 'http://127.0.0.1:6333'; // Or your Qdrant URL
       private $embeddingModel;
       private $gptModel;
       private $embeddingDims;

       public function __construct()
       {
           $this->openaiApiKey = env('OPENAI_API_KEY');
           $this->embeddingModel = env('EMBEDDING_MODEL', 'text-embedding-ada-002');
           $this->gptModel = env('GPT_MODEL', 'gpt-4o-mini');
           $this->embeddingDims = env('EMBEDDING_MODEL_DIMS', 1536);
       }

       // Helper: Call OpenAI API
       private function callOpenAI($endpoint, $data)
       {
           $response = Http::withHeaders([
               'Authorization' => 'Bearer ' . $this->openaiApiKey,
               'Content-Type' => 'application/json',
           ])->post("https://api.openai.com/v1/$endpoint", $data);

           if ($response->failed()) {
               throw new \Exception('OpenAI API error: ' . $response->body());
           }

           return $response->json();
       }

       // Get embedding vector for text
       public function getEmbedding($text)
       {
           $data = [
               'model' => $this->embeddingModel,
               'input' => $text,
           ];
           $response = $this->callOpenAI('embeddings', $data);
           return $response['data'][0]['embedding'];
       }

       // Qdrant: Check if collection exists
       public function qdrantCollectionExists($collectionName)
       {
           $response = Http::get("{$this->qdrantEndpoint}/collections/{$collectionName}/exists");
           return $response->json()['result']['exists'] ?? false;
       }

       // Qdrant: Create collection
       public function qdrantCreateCollection($collectionName)
       {
           $response = Http::put("{$this->qdrantEndpoint}/collections/{$collectionName}", [
               'vectors' => [
                   'size' => $this->embeddingDims,
                   'distance' => 'Cosine',
               ],
           ]);
           return $response->successful();
       }

       // RAG: Add document to vector store
       public function addDocumentToRAG($collectionName, $docId, $text)
       {
           if (!$this->qdrantCollectionExists($collectionName)) {
               $this->qdrantCreateCollection($collectionName);
           }

           $vector = $this->getEmbedding($text);
           $response = Http::put("{$this->qdrantEndpoint}/collections/{$collectionName}/points", [
               'batch' => [
                   'ids' => [$docId],
                   'vectors' => [$vector],
                   'payloads' => [['text' => $text]],
               ],
           ]);

           return $response->successful();
       }

       // RAG: Retrieve context for query
       public function retrieveContext($collectionName, $query, $limit = 5)
       {
           if (!$this->qdrantCollectionExists($collectionName)) {
               return '';
           }

           $vector = $this->getEmbedding($query);
           $response = Http::post("{$this->qdrantEndpoint}/collections/{$collectionName}/points/search", [
               'vector' => $vector,
               'limit' => $limit,
               'with_payload' => true,
           ]);

           $results = $response->json()['result'] ?? [];
           $context = '';
           foreach ($results as $result) {
               if ($result['score'] > 0.7) { // Threshold for relevance
                   $context .= $result['payload']['text'] . "\n\n";
               }
           }
           return trim($context);
       }

       // Define tools (example: weather and web search)
       private function getTools()
       {
           return [
               [
                   'type' => 'function',
                   'function' => [
                       'name' => 'get_current_weather',
                       'description' => 'Get current weather for a location',
                       'parameters' => [
                           'type' => 'object',
                           'properties' => [
                               'location' => ['type' => 'string', 'description' => 'City name'],
                           ],
                           'required' => ['location'],
                       ],
                   ],
               ],
               [
                   'type' => 'function',
                   'function' => [
                       'name' => 'search_web',
                       'description' => 'Search the web for information',
                       'parameters' => [
                           'type' => 'object',
                           'properties' => [
                               'query' => ['type' => 'string', 'description' => 'Search query'],
                           ],
                           'required' => ['query'],
                       ],
                   ],
               ],
           ];
       }

       // Execute a tool based on name and args
       private function executeTool($toolName, $arguments)
       {
           $arguments = json_decode($arguments, true);

           switch ($toolName) {
               case 'get_current_weather':
                   // Mock or call real API (e.g., weather API)
                   return "Weather in {$arguments['location']}: Sunny, 25°C";
               case 'search_web':
                   // Mock or call search API (e.g., Google Custom Search)
                   return "Search results for '{$arguments['query']}': Example result 1, Example result 2.";
               default:
                   return 'Tool not found';
           }
       }

       // Main agent function: Handle query with RAG and tools
       public function processQuery($query, $collectionName = 'default')
       {
           $messages = [['role' => 'user', 'content' => $query]];
           $context = $this->retrieveContext($collectionName, $query);
           $systemPrompt = "You are an AI agent. Use provided context if relevant. Context: $context";

           $maxIterations = 5; // Prevent infinite loops
           $iteration = 0;

           while ($iteration < $maxIterations) {
               $data = [
                   'model' => $this->gptModel,
                   'messages' => array_merge([['role' => 'system', 'content' => $systemPrompt]], $messages),
                   'tools' => $this->getTools(),
                   'tool_choice' => 'auto',
               ];

               $response = $this->callOpenAI('chat/completions', $data);
               $choice = $response['choices'][0];
               $message = $choice['message'];

               if (isset($message['content'])) {
                   return $message['content']; // Final response
               }

               if (isset($message['tool_calls'])) {
                   foreach ($message['tool_calls'] as $toolCall) {
                       $toolName = $toolCall['function']['name'];
                       $args = $toolCall['function']['arguments'];
                       $toolResult = $this->executeTool($toolName, $args);
                       $messages[] = $message; // Add assistant's tool call
                       $messages[] = [
                           'role' => 'tool',
                           'tool_call_id' => $toolCall['id'],
                           'name' => $toolName,
                           'content' => $toolResult,
                       ];
                   }
               }

               $iteration++;
           }

           return 'Max iterations reached. Try simplifying your query.';
       }
   }
   ```

5. Create an API controller:
   ```
   php artisan make:controller Api/AgentController
   ```

   In `app/Http/Controllers/Api/AgentController.php`:
   ```php
   <?php

   namespace App\Http\Controllers\Api;

   use App\Http\Controllers\Controller;
   use App\Services\AIAgentService;
   use Illuminate\Http\Request;

   class AgentController extends Controller
   {
       public function handleQuery(Request $request, AIAgentService $service)
       {
           $query = $request->input('query');
           $collectionName = $request->input('collection', 'default'); // For RAG
           $response = $service->processQuery($query, $collectionName);
           return response()->json(['response' => $response]);
       }

       public function addDocument(Request $request, AIAgentService $service)
       {
           $collectionName = $request->input('collection', 'default');
           $docId = Str::uuid();
           $text = $request->input('text');
           $success = $service->addDocumentToRAG($collectionName, $docId, $text);
           return response()->json(['success' => $success]);
       }
   }
   ```

6. Define routes in `routes/api.php`:
   ```php
   Route::post('/agent/query', [AgentController::class, 'handleQuery']);
   Route::post('/agent/add-document', [AgentController::class, 'addDocument']);
   ```

### Step 2: Set Up React Vite Frontend
1. Install frontend dependencies:
   ```
   npm install
   npm install --save-dev @vitejs/plugin-react
   ```

2. Update `vite.config.js`:
   ```javascript
   import { defineConfig } from 'vite';
   import laravel from 'laravel-vite-plugin';
   import react from '@vitejs/plugin-react';

   export default defineConfig({
       plugins: [
           laravel(['resources/js/app.jsx']),
           react(),
       ],
   });
   ```

3. Create `resources/js/app.jsx` (basic chat interface):
   ```jsx
   import React, { useState } from 'react';
   import ReactDOM from 'react-dom/client';

   function App() {
       const [query, setQuery] = useState('');
       const [response, setResponse] = useState('');
       const [collection, setCollection] = useState('default');
       const [docText, setDocText] = useState('');

       const handleQuery = async () => {
           const res = await fetch('/api/agent/query', {
               method: 'POST',
               headers: { 'Content-Type': 'application/json' },
               body: JSON.stringify({ query, collection }),
           });
           const data = await res.json();
           setResponse(data.response);
       };

       const addDocument = async () => {
           await fetch('/api/agent/add-document', {
               method: 'POST',
               headers: { 'Content-Type': 'application/json' },
               body: JSON.stringify({ collection, text: docText }),
           });
           setDocText('');
       };

       return (
           <div>
               <h1>AI Agent Chat</h1>
               <input value={query} onChange={e => setQuery(e.target.value)} placeholder="Ask something..." />
               <button onClick={handleQuery}>Send</button>
               <p>Response: {response}</p>
               <hr />
               <h2>Add Document to RAG</h2>
               <textarea value={docText} onChange={e => setDocText(e.target.value)} />
               <input value={collection} onChange={e => setCollection(e.target.value)} placeholder="Collection name" />
               <button onClick={addDocument}>Add</button>
           </div>
       );
   }

   ReactDOM.createRoot(document.getElementById('root')).render(<App />);
   ```

4. In `resources/views/welcome.blade.php` (or your main view):
   ```blade
   <!DOCTYPE html>
   <html>
   <head>
       @viteReactRefresh
       @vite('resources/js/app.jsx')
   </head>
   <body>
       <div id="root"></div>
   </body>
   </html>
   ```

5. Run development servers:
   - Backend: `php artisan serve`
   - Frontend: `npm run dev`

### Step 3: Usage and Extension
- **Add documents**: Use the `/agent/add-document` endpoint to index text into RAG (e.g., via frontend or curl).
- **Query the agent**: Send queries to `/agent/query`. The agent retrieves RAG context, calls LLM, handles tools in a loop if needed.
- **Extend tools**: Add more to `getTools()` and `executeTool()` (e.g., integrate real APIs).
- **Security**: Add authentication (e.g., Sanctum) for production.
- **Testing**: Test RAG by adding docs and querying related topics. For tools, ask something like "What's the weather in London?".

This setup provides a basic, framework-free AI agent. Scale by adding more tools or using a production vector DB.
