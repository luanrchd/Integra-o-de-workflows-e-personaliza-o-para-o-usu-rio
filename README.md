

```php
<?php // database/migrations/xxxx_xx_xx_create_ai_personas_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('ai_personas', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('name'); // e.g., "Formal Email Responder"
            $table->text('instructions'); // The core prompt/instructions for this persona
            $table->json('examples')->nullable(); // Optional: Few-shot examples [{input: '', output: ''}]
            $table->boolean('is_default')->default(false);
            $table->timestamps();

            $table->unique(['user_id', 'name']);
        });
    }
    // down() method...
};

// ---

<?php // app/Models/AiPersona.php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class AiPersona extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'name',
        'instructions',
        'examples',
        'is_default',
    ];

    protected $casts = [
        'examples' => 'array',
        'is_default' => 'boolean',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    // Scope to get the default persona or the first one if none is default
    public function scopeDefaultOrFirst(Builder $query, int $userId): Builder
    {
        return $query->where('user_id', $userId)
                     ->orderBy('is_default', 'desc') // Prioritize default
                     ->orderBy('created_at', 'asc');   // Fallback to oldest
    }
}

// ---

<?php // app/Services/AiInteractionService.php

namespace App\Services;

use App\Models\AiPersona;
use App\Models\User;
use OpenAI\Client as OpenAIClient; // Example using OpenAI client
use Illuminate\Support\Facades\Log;

class AiInteractionService
{
    protected OpenAIClient $openai;

    public function __construct(OpenAIClient $openai)
    {
        $this->openai = $openai;
    }

    /**
     * Process a task using the AI model, incorporating user context and persona.
     *
     * @param string $taskType Example: "summarize", "draft_email", "explain"
     * @param string $inputData The main input (e.g., text to summarize)
     * @param User $user The user requesting the action
     * @param int|null $personaId The specific persona ID to use (optional)
     * @param array $additionalContext Extra context (e.g., from browser extension)
     * @return string The AI-generated result
     * @throws \Exception
     */
    public function processTask(string $taskType, string $inputData, User $user, ?int $personaId = null, array $additionalContext = []): string
    {
        $persona = $this->loadPersona($user, $personaId);

        $systemPrompt = $this->buildSystemPrompt($persona, $taskType, $additionalContext);
        $userPrompt = $this->buildUserPrompt($inputData, $additionalContext);

        Log::debug("AI Request - System Prompt: " . $systemPrompt);
        Log::debug("AI Request - User Prompt: " . $userPrompt);

        try {
            // Example using OpenAI Chat Completions API
            $response = $this->openai->chat()->create([
                'model' => config('services.openai.model', 'gpt-4o'), // Use configurable model
                'messages' => [
                    ['role' => 'system', 'content' => $systemPrompt],
                    ['role' => 'user', 'content' => $userPrompt],
                ],
                'temperature' => 0.7, // Adjust as needed
                // Add other parameters like max_tokens, etc.
                'user' => 'user-' . $user->id, // Optional: Pass user ID for monitoring abuse
            ]);

            $result = $response->choices[0]->message->content ?? '';

            // Basic logging of usage (a more robust system would track tokens)
            Log::info("AI Task Completed for User {$user->id}. Task: {$taskType}. Persona: {$persona?->name ?? 'Default'}.");

            return trim($result);

        } catch (\Exception $e) {
            Log::error("AI Interaction Failed for User {$user->id}: " . $e->getMessage());
            // Re-throw or return a user-friendly error message
            throw new \Exception("Falha ao processar a solicitação com a IA. Tente novamente mais tarde.");
        }
    }

    protected function loadPersona(User $user, ?int $personaId): ?AiPersona
    {
        if ($personaId) {
            return AiPersona::where('user_id', $user->id)->findOrFail($personaId);
        }
        // Load default or first available persona for the user
        return AiPersona::defaultOrFirst($user->id)->first();
    }

    protected function buildSystemPrompt(?AiPersona $persona, string $taskType, array $context): string
    {
        $prompt = "Você é um assistente de IA da plataforma Ovyva, ajudando usuários a economizar tempo.\n";

        if ($persona) {
            $prompt .= "Siga estas instruções de persona:\n--- INSTRUÇÕES DA PERSONA ---\n";
            $prompt .= $persona->instructions;
            $prompt .= "\n--- FIM DAS INSTRUÇÕES ---\n\n";

            if (!empty($persona->examples)) {
                $prompt .= "Use estes exemplos como guia:\n";
                foreach ($persona->examples as $example) {
                    $prompt .= "Exemplo Entrada: {$example['input']}\nExemplo Saída: {$example['output']}\n---\n";
                }
                $prompt .= "\n";
            }
        } else {
            $prompt .= "Aja como um assistente geral prestativo.\n";
        }

        // Add task-specific instructions based on $taskType
        switch ($taskType) {
            case 'summarize':
                $prompt .= "Sua tarefa atual é SUMARIZAR o texto fornecido pelo usuário.";
                break;
            case 'draft_email_reply':
                $prompt .= "Sua tarefa atual é RASCUNHAR uma RESPOSTA DE EMAIL com base no email original e nas notas do usuário.";
                break;
            case 'explain':
                 $prompt .= "Sua tarefa atual é EXPLICAR o conceito ou texto fornecido pelo usuário de forma clara.";
                 break;
            // Add more task types
            default:
                 $prompt .= "Sua tarefa atual é processar a entrada do usuário conforme solicitado.";
        }

        // Add context from browser extension if available
        if (!empty($context['pageTitle'])) {
            $prompt .= "\nContexto adicional: O usuário está na página '{$context['pageTitle']}'.";
        }
        if (!empty($context['pageUrl'])) {
            $prompt .= "\nURL da página: {$context['pageUrl']}";
        }


        $prompt .= "\nResponda de forma concisa e direta ao ponto, a menos que a persona instrua o contrário.";
        return $prompt;
    }

     protected function buildUserPrompt(string $inputData, array $context): string
     {
         // Depending on the task, the user prompt might need structuring
         // Example for email reply
         if (!empty($context['originalEmail'])) {
             return "Email Original:\n```\n" . $context['originalEmail'] . "\n```\n\nNotas para a resposta:\n```\n" . $inputData . "\n```\n\nRascunhe a resposta:";
         }

         // Default: just use the input data
         return $inputData;
     }
}


// ---

<?php // app/Http/Controllers/Api/AiActionController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\AiActionRequest; // Custom Form Request for validation
use App\Services\AiInteractionService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request; // Use Illuminate\Http\Request
use Illuminate\Support\Facades\Auth;

class AiActionController extends Controller
{
    protected AiInteractionService $aiService;

    public function __construct(AiInteractionService $aiService)
    {
        $this->aiService = $aiService;
    }

    /**
     * Handle incoming AI action requests (e.g., from browser extension or frontend).
     */
    public function process(AiActionRequest $request): JsonResponse // Use Form Request
    {
        try {
            $validated = $request->validated();
            /** @var User $user */
            $user = Auth::user(); // Assumes authenticated via Sanctum

            $result = $this->aiService->processTask(
                $validated['task_type'],       // e.g., 'summarize'
                $validated['input_data'],      // e.g., selected text
                $user,
                $validated['persona_id'] ?? null, // Optional persona ID
                $validated['context'] ?? []     // Optional context (page title, URL, etc.)
            );

            return response()->json(['result' => $result]);

        } catch (\Exception $e) {
            // Log the full error internally if needed
            return response()->json(['error' => $e->getMessage()], 500);
        }
    }
}

// ---

<?php // app/Http/Requests/AiActionRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class AiActionRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Only authenticated users can perform AI actions
        return $this->user() != null;
    }

    public function rules(): array
    {
        return [
            'task_type' => ['required', 'string', Rule::in(['summarize', 'draft_email_reply', 'explain', 'translate', /* add more allowed tasks */])],
            'input_data' => 'required|string|max:10000', // Limit input size
            'persona_id' => 'nullable|integer|exists:ai_personas,id,user_id,' . $this->user()?->id, // Must exist and belong to user
            'context' => 'nullable|array',
            'context.pageTitle' => 'nullable|string|max:255',
            'context.pageUrl' => 'nullable|string|url|max:2048',
            'context.originalEmail' => 'nullable|string|max:5000', // Example specific context
            // Add validation for other context fields if needed
        ];
    }
}

// ---

// routes/api.php - API route protected by Sanctum
use App\Http\Controllers\Api\AiActionController;
use App\Http\Controllers\Api\AiPersonaController; // Assuming a CRUD controller for personas

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/ai-action', [AiActionController::class, 'process']);

    // CRUD for AI Personas
    Route::apiResource('/ai-personas', AiPersonaController::class);
    // Route::post('/ai-personas/{persona}/set-default', [AiPersonaController::class, 'setDefault']); // Example custom action
});
```



```jsx
// src/components/AiPersonaManager.jsx (Simplified Example)
import React, { useState, useEffect, useCallback } from 'react';
import apiClient from '../services/apiClient'; // Your configured Axios instance

function AiPersonaManager() {
  const [personas, setPersonas] = useState([]);
  const [currentPersona, setCurrentPersona] = useState({ id: null, name: '', instructions: '', examples: [] });
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState('');

  const fetchPersonas = useCallback(async () => {
    setIsLoading(true);
    setError('');
    try {
      const response = await apiClient.get('/ai-personas');
      setPersonas(response.data.data); // Assuming paginated or resource collection response
    } catch (err) {
      setError('Falha ao carregar personas.');
      console.error(err);
    } finally {
      setIsLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchPersonas();
  }, [fetchPersonas]);

  const handleSelectPersona = (persona) => {
    setCurrentPersona({ ...persona, examples: persona.examples || [] }); // Ensure examples is an array
  };

  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setCurrentPersona(prev => ({ ...prev, [name]: value }));
  };

  // Basic example handling - a real implementation would be more robust
  const handleExampleChange = (index, field, value) => {
      const updatedExamples = [...currentPersona.examples];
      if (!updatedExamples[index]) updatedExamples[index] = { input: '', output: ''};
      updatedExamples[index][field] = value;
      setCurrentPersona(prev => ({ ...prev, examples: updatedExamples }));
  }
   const addExample = () => {
        setCurrentPersona(prev => ({ ...prev, examples: [...prev.examples, { input: '', output: '' }]}));
    }

  const handleSave = async (e) => {
    e.preventDefault();
    setIsLoading(true);
    setError('');
    const { id, ...data } = currentPersona;
    const url = id ? `/ai-personas/${id}` : '/ai-personas';
    const method = id ? 'put' : 'post';

    try {
      const response = await apiClient[method](url, data);
      alert('Persona salva com sucesso!');
      fetchPersonas(); // Refresh list
      // Optionally reset form or update currentPersona with response
      setCurrentPersona({ id: null, name: '', instructions: '', examples: [] });
    } catch (err) {
      setError(err.response?.data?.message || 'Erro ao salvar persona.');
      console.error(err);
    } finally {
      setIsLoading(false);
    }
  };

   const handleDelete = async (personaId) => {
        if (!personaId || !window.confirm('Tem certeza que deseja excluir esta persona?')) return;
        setIsLoading(true);
        setError('');
        try {
            await apiClient.delete(`/ai-personas/${personaId}`);
            alert('Persona excluída!');
            fetchPersonas();
             if(currentPersona.id === personaId) {
                 setCurrentPersona({ id: null, name: '', instructions: '', examples: [] });
             }
        } catch (err) {
            setError(err.response?.data?.message || 'Erro ao excluir persona.');
            console.error(err);
        } finally {
            setIsLoading(false);
        }
    };


  return (
    <div className="persona-manager">
      <h2>Gerenciar Personas de IA</h2>
      {error && <p className="error">{error}</p>}
      {isLoading && <p>Carregando...</p>}

      <div className="persona-list">
        <h3>Suas Personas</h3>
        <ul>
          {personas.map(p => (
            <li key={p.id}>
              {p.name} {p.is_default ? '(Padrão)' : ''}
              <button onClick={() => handleSelectPersona(p)}>Editar</button>
              <button onClick={() => handleDelete(p.id)} disabled={isLoading}>Excluir</button>
            </li>
          ))}
        </ul>
        <button onClick={() => setCurrentPersona({ id: null, name: '', instructions: '', examples: [] })}>
            + Nova Persona
        </button>
      </div>

      <form onSubmit={handleSave} className="persona-form">
        <h3>{currentPersona.id ? 'Editar Persona' : 'Nova Persona'}</h3>
        <div>
          <label htmlFor="name">Nome:</label>
          <input
            type="text"
            id="name"
            name="name"
            value={currentPersona.name}
            onChange={handleInputChange}
            required
            maxLength="255"
          />
        </div>
        <div>
          <label htmlFor="instructions">Instruções:</label>
          <textarea
            id="instructions"
            name="instructions"
            value={currentPersona.instructions}
            onChange={handleInputChange}
            rows="10"
            required
          />
        </div>
         <div>
            <h4>Exemplos (Opcional):</h4>
            {currentPersona.examples.map((ex, index) => (
                <div key={index} className="persona-example">
                    <label>Entrada {index+1}:</label>
                    <textarea value={ex.input} onChange={(e) => handleExampleChange(index, 'input', e.target.value)} />
                    <label>Saída {index+1}:</label>
                    <textarea value={ex.output} onChange={(e) => handleExampleChange(index, 'output', e.target.value)} />
                </div>
            ))}
             <button type="button" onClick={addExample}>+ Adicionar Exemplo</button>
        </div>
        {/* Add checkbox for is_default if needed */}
        <button type="submit" disabled={isLoading}>
          {isLoading ? 'Salvando...' : 'Salvar Persona'}
        </button>
      </form>
    </div>
  );
}

export default AiPersonaManager;
```


```javascript
// manifest.json (Simplified)
{
  "manifest_version": 3,
  "name": "Ovyva AI Helper",
  "version": "1.0",
  "description": "Use Ovyva AI em qualquer lugar na web.",
  "permissions": [
    "contextMenus", // To add right-click menu items
    "storage",      // To store user token, settings
    "activeTab",    // To get page context like title/url if needed
    "scripting"     // To inject content scripts if needed (less common with manifest v3)
    // "cookies"    // Needed to access cookies for your API domain (use carefully)
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"], // Run on all pages
      "js": ["content.js"]
    }
  ],
   "action": { // Popup when clicking the extension icon
     "default_popup": "popup.html",
     "default_icon": "icon16.png"
   },
  "host_permissions": [
    "https://sua-api-ovyva.com/*" // IMPORTANT: Allow connection to your API backend
  ],
   "icons": { // Example icons
       "16": "icon16.png",
       "48": "icon48.png",
       "128": "icon128.png"
   }
}

// ---

// background.js (Service Worker)

const API_URL = 'https://sua-api-ovyva.com/api'; // Your Laravel API endpoint

// Function to get auth token (implement proper storage)
async function getToken() {
  // IMPORTANT: Implement secure token storage (chrome.storage.local or session)
  // For simplicity, using local storage here, but not recommended for sensitive tokens.
  const result = await chrome.storage.local.get(['authToken']);
  return result.authToken;
}


// --- Context Menu Setup ---
chrome.runtime.onInstalled.addListener(() => {
  chrome.contextMenus.create({
    id: "ovyva_summarize",
    title: "Ovyva: Sumarizar seleção",
    contexts: ["selection"] // Show only when text is selected
  });
   chrome.contextMenus.create({
    id: "ovyva_explain",
    title: "Ovyva: Explicar seleção",
    contexts: ["selection"]
  });
  // Add more context menu items for different tasks...
});

// --- Context Menu Click Handler ---
chrome.contextMenus.onClicked.addListener(async (info, tab) => {
  if (!info.selectionText) {
    console.warn("No text selected.");
    return;
  }

  const taskMap = {
      "ovyva_summarize": "summarize",
      "ovyva_explain": "explain",
      // ... map other context menu IDs to task_type
  };

  const taskType = taskMap[info.menuItemId];
  if (!taskType) return;

  const authToken = await getToken();
  if (!authToken) {
    // Handle not logged in - maybe open login page or popup
    console.error("Ovyva: User not authenticated.");
    // chrome.runtime.sendMessage({ type: 'SHOW_LOGIN_REQUIRED' }); // Send message to content/popup script
    return;
  }

  try {
     // Show some loading indicator to the user (e.g., via content script message)
     chrome.runtime.sendMessage({ type: 'OVYVA_LOADING' }); // Send message to content script

    const response = await fetch(`${API_URL}/ai-action`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': `Bearer ${authToken}` // Send auth token
      },
      body: JSON.stringify({
        task_type: taskType,
        input_data: info.selectionText,
        // persona_id: await getSelectedPersonaId(), // Fetch selected persona from storage
        context: { // Get context from the tab (requires activeTab permission or messaging)
           pageTitle: tab.title,
           pageUrl: tab.url
        }
      })
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.error || `HTTP error! status: ${response.status}`);
    }

    // Send result back to content script or display in popup/notification
     chrome.runtime.sendMessage({ type: 'OVYVA_RESULT', payload: data.result });

  } catch (error) {
    console.error(`Ovyva AI Action Error (${taskType}):`, error);
     chrome.runtime.sendMessage({ type: 'OVYVA_ERROR', payload: error.message });
  }
});

// --- Message Listener (for communication with content/popup scripts) ---
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'GET_USER_STATUS') {
     // Check token and respond if user is logged in
     getToken().then(token => sendResponse({ loggedIn: !!token }));
     return true; // Indicates async response
  }
  // Handle other messages (e.g., trigger actions from popup)
});


// ---

// content.js (Content Script - Basic Example for showing results)

// --- Listener for messages from background script ---
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if (message.type === 'OVYVA_RESULT') {
        // Display the result (e.g., in a temporary overlay, console, or alert)
        console.log("Ovyva Result:", message.payload);
        alert("Ovyva Result:\n\n" + message.payload); // Simple alert for demo
        // A better implementation would inject a styled overlay near the selection
        hideLoadingIndicator();
    } else if (message.type === 'OVYVA_ERROR') {
        console.error("Ovyva Error:", message.payload);
        alert("Ovyva Error:\n\n" + message.payload);
         hideLoadingIndicator();
    } else if (message.type === 'OVYVA_LOADING') {
        showLoadingIndicator();
    }
});


// --- Basic Loading/Result Display (very rudimentary) ---
function showLoadingIndicator() {
    let indicator = document.getElementById('ovyva-indicator');
    if (!indicator) {
        indicator = document.createElement('div');
        indicator.id = 'ovyva-indicator';
        indicator.style.position = 'fixed';
        indicator.style.top = '10px';
        indicator.style.right = '10px';
        indicator.style.padding = '10px';
        indicator.style.backgroundColor = 'rgba(0,0,0,0.7)';
        indicator.style.color = 'white';
        indicator.style.zIndex = '99999';
        indicator.style.borderRadius = '5px';
        document.body.appendChild(indicator);
    }
    indicator.textContent = 'Ovyva processando...';
     indicator.style.display = 'block';
}

function hideLoadingIndicator() {
     let indicator = document.getElementById('ovyva-indicator');
     if (indicator) {
         indicator.style.display = 'none';
     }
}

// Optional: Add listeners for user selection changes if needed for dynamic UI
```

