<script lang="ts">
  import { onMount, tick } from 'svelte';
  import { marked } from 'marked'; // npm install marked

  // API keys from environment variables (declare once at the top)
  const apiKeyForGemini = import.meta.env.VITE_GEMINI_API_KEY;
  const apiKeyForGoogleMaps = import.meta.env.VITE_GOOGLE_MAPS_API_KEY;

  let currentScreen: 'home' | 'chat' | 'map' | 'planner' = 'home';

  // Gemini API integration
  type ChatMessage = { role: 'user' | 'assistant'; content: string; markdown?: boolean };
  let chatInput = '';
  let chatMessages: ChatMessage[] = [];
  let isLoading = false;

  async function sendMessage() {
    if (!chatInput.trim()) return;
    chatMessages = [...chatMessages, { role: 'user', content: chatInput }];
    const userMessage = chatInput;
    chatInput = '';
    isLoading = true;
    try {
      const endpoint = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=' + apiKeyForGemini;
      const res = await fetch(endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          systemInstruction: {
            parts: [{ text: "If you provide pins for recommended venues, always include a JSON array in a code block at the last part of your answer. The JSON should be an array of objects with keys: label (or name), lat (or latitude), lng (or longitude), and optionally icon and description. The rest of your answer should be a natural language explanation in markdown."}]
          },
          contents: [{ parts: [{ text: userMessage }] }]
        })
      });
      const data = await res.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text || '[No response]';
      // Extract JSON code block and markdown part
      const codeBlockRegex = /```json[\s\r\n]*([\s\S]*?)```/;
      const match = text.match(codeBlockRegex);
      let markdownPart = text;
      let jsonText = null;
      if (match) {
        markdownPart = text.replace(codeBlockRegex, '').trim(); // Remove code block for chat display
        jsonText = match[1];
      }
      // Show markdownPart in chat (rendered as markdown)
      chatMessages = [...chatMessages, { role: 'assistant', content: markdownPart, markdown: true }];
      // Use jsonText for pins (as before)
      if (jsonText) {
        try {
          const locations = JSON.parse(jsonText);
          if (Array.isArray(locations) && (locations[0]?.lat && locations[0]?.lng || locations[0]?.latitude && locations[0]?.longitude)) {
            // Map Gemini's keys to your expected keys
            const pins = locations.map((loc: any) => ({
              lat: loc.lat ?? loc.latitude,
              lng: loc.lng ?? loc.longitude,
              label: loc.label ?? loc.name,
              icon: loc.icon,
              description: loc.description
            }));
            setMapPins(pins);
          }
        } catch {}
      }
    } catch (e) {
      chatMessages = [...chatMessages, { role: 'assistant', content: '[Error contacting Gemini API]' }];
    }
    isLoading = false;
  }

  // Google Maps integration
  let map: any = undefined;
  let mapElement: HTMLDivElement | null = null;
  let mapPins: {lat: number, lng: number, label?: string}[] = [];
  let mapMarkers: any[] = [];
  let mapInitialized = false;

  function loadGoogleMaps() {
    return new Promise<any>((resolve, reject) => {
      if ((window as any).google && (window as any).google.maps) return resolve((window as any).google.maps);
      // Prevent duplicate script injection
      if (document.querySelector('script[data-google-maps]')) {
        // Wait for it to finish loading
        (window as any).gmapsLoaded
          ? resolve((window as any).google.maps)
          : window.addEventListener('gmaps:loaded', () => resolve((window as any).google.maps));
        return;
      }
      const script = document.createElement('script');
      script.src = `https://maps.googleapis.com/maps/api/js?key=${apiKeyForGoogleMaps}`;
      script.async = true;
      script.defer = true;
      script.setAttribute('data-google-maps', 'true');
      script.onload = () => {
        (window as any).gmapsLoaded = true;
        window.dispatchEvent(new Event('gmaps:loaded'));
        resolve((window as any).google.maps);
      };
      script.onerror = reject;
      document.body.appendChild(script);
    });
  }

  function updateMapPins() {
    if (!map) return;
    // Remove old markers
    mapMarkers.forEach(m => m.setMap(null));
    mapMarkers = [];
    // Add new markers
    mapPins.forEach(pin => {
      const marker = new (window as any).google.maps.Marker({
        position: { lat: pin.lat, lng: pin.lng },
        map: map,
        label: pin.label || undefined
      });
      mapMarkers.push(marker);
    });
  }

  function setMapPins(newPins: {lat: number, lng: number, label?: string}[]) {
    mapPins = newPins;
  }

  $: if (map && currentScreen === 'map') updateMapPins();

  $: if (currentScreen === 'map' && !mapInitialized) {
    tick().then(async () => {
      if (mapElement && !mapInitialized) {
        const gmaps = await loadGoogleMaps();
        map = new gmaps.Map(mapElement, {
          center: { lat: 37.5665, lng: 126.9780 },
          zoom: 10
        });
        updateMapPins();
        mapInitialized = true;
      }
    });
  }

  $: if (currentScreen !== 'map') {
    mapInitialized = false;
  }
</script>

<nav>
  <button class:active={currentScreen === 'home'} on:click={() => currentScreen = 'home'}>Home</button>
  <button class:active={currentScreen === 'chat'} on:click={() => currentScreen = 'chat'}>Chatbot</button>
  <button class:active={currentScreen === 'planner'} on:click={() => currentScreen = 'planner'}>Itinerary Planner</button>
  <button class:active={currentScreen === 'map'} on:click={() => currentScreen = 'map'}>Itinerary Map</button>
</nav>

<main>
  {#if currentScreen === 'home'}
    <section class="hero">
      <h1>🌍 Postport</h1>
      <p>Your smart travel assistant. Plan, chat, and explore with AI-powered ease!</p>
      <img src="https://images.unsplash.com/photo-1506744038136-46273834b3fb?auto=format&fit=crop&w=800&q=80" alt="Travel" class="hero-img" />
    </section>
  {:else if currentScreen === 'chat'}
    <section class="feature">
      <h2>💬 Chatbot Assistant</h2>
      <div class="chat-window">
        {#each chatMessages as msg}
          <div class="msg {msg.role}">
            {msg.role === 'user' ? '🧑' : '🤖'}
            {#if msg.markdown}
              <span>{@html marked.parse(msg.content)}</span>
            {:else}
              {msg.content}
            {/if}
          </div>
        {/each}
        {#if isLoading}
          <div class="msg assistant">🤖 ...</div>
        {/if}
      </div>
      <form class="chat-input" on:submit|preventDefault={sendMessage}>
        <input type="text" bind:value={chatInput} placeholder="Ask Gemini..." autocomplete="off" />
        <button type="submit" disabled={isLoading || !chatInput.trim()}>Send</button>
      </form>
      <div class="hint">
        Tip: Give it to Gemini that your favor of tour style specifically as much as you can!<br>
        Example: <code>Tell me about 5 must-see venues in Japan for the "Bocchi the Rock" lovers.</code>
      </div>
    </section>
  {:else if currentScreen === 'planner'}
    <section class="feature">
      <h2>📝 Itinerary Planner</h2>
      <div class="placeholder">[Itinerary Planner UI coming soon]</div>
    </section>
  {:else if currentScreen === 'map'}
    <section class="feature">
      <h2>🗺️ Itinerary Map</h2>
      <div bind:this={mapElement} class="map-view"></div>
      {#if mapPins.length === 0}
        <div class="placeholder">No pins yet. Ask Gemini for places in chat!</div>
      {/if}
    </section>
  {/if}
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
  html, body {
    font-family: 'Inter', system-ui, sans-serif;
    background: linear-gradient(135deg, #f8fafc 0%, #e0e7ff 100%);
    min-height: 100vh;
    color: #222;
  }
  nav {
    display: flex;
    justify-content: center;
    gap: 1.5em;
    background: rgba(255,255,255,0.55);
    backdrop-filter: blur(12px);
    padding: 1.2em 0 1em 0;
    border-radius: 0 0 32px 32px;
    box-shadow: 0 4px 32px 0 rgba(80,80,120,0.08);
    margin-bottom: 2.5em;
    position: sticky;
    top: 0;
    z-index: 10;
  }
  nav button {
    background: linear-gradient(90deg, #ff9800 0%, #ff3e00 100%);
    color: #fff;
    font-size: 1.08em;
    font-weight: 600;
    padding: 0.6em 1.5em;
    border-radius: 22px;
    border: none;
    cursor: pointer;
    box-shadow: 0 2px 8px rgba(255,152,0,0.08);
    transition: background 0.18s, transform 0.13s, box-shadow 0.18s;
    outline: none;
  }
  nav button.active, nav button:hover {
    background: linear-gradient(90deg, #ffb347 0%, #ff3e00 100%);
    transform: translateY(-2px) scale(1.04);
    box-shadow: 0 4px 16px rgba(255,152,0,0.13);
  }
  main {
    max-width: 720px;
    margin: 0 auto;
    padding: 2.5em 1.2em 2em 1.2em;
    text-align: center;
  }
  .hero {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.7em;
    background: rgba(255,255,255,0.65);
    backdrop-filter: blur(18px);
    border-radius: 32px;
    box-shadow: 0 8px 40px 0 rgba(255,152,0,0.10);
    padding: 2.5em 1.2em 2.2em 1.2em;
    margin-bottom: 2em;
    border: 1.5px solid rgba(255,152,0,0.08);
  }
  .hero-img {
    width: 100%;
    max-width: 420px;
    border-radius: 22px;
    box-shadow: 0 4px 24px rgba(0,0,0,0.10);
    object-fit: cover;
  }
  .feature {
    background: rgba(255,255,255,0.72);
    backdrop-filter: blur(16px);
    border-radius: 24px;
    box-shadow: 0 6px 32px 0 rgba(33, 150, 243, 0.10);
    padding: 2.2em 1.2em 2em 1.2em;
    margin-bottom: 2.5em;
    border: 1.5px solid rgba(33,150,243,0.07);
    position: relative;
  }
  .placeholder {
    margin-top: 2.2em;
    color: #b0b0b0;
    font-size: 1.18em;
    font-style: italic;
    background: rgba(240,245,255,0.7);
    border-radius: 16px;
    padding: 2.2em 0;
    box-shadow: 0 2px 12px rgba(33, 150, 243, 0.04);
  }
  .chat-window {
    background: rgba(245,245,255,0.85);
    border-radius: 16px;
    padding: 1.2em 1em 1em 1em;
    min-height: 180px;
    max-height: 320px;
    overflow-y: auto;
    margin-bottom: 1.2em;
    text-align: left;
    box-shadow: 0 2px 12px rgba(33, 150, 243, 0.06);
    border: 1.2px solid rgba(33,150,243,0.06);
    display: flex;
    flex-direction: column;
    gap: 0.5em;
  }
  .msg {
    margin-bottom: 0.2em;
    padding: 0.7em 1.2em;
    border-radius: 14px;
    max-width: 85%;
    word-break: break-word;
    font-size: 1.04em;
    box-shadow: 0 2px 8px rgba(33,150,243,0.04);
    display: inline-block;
    line-height: 1.6;
    position: relative;
    animation: fadeInMsg 0.4s cubic-bezier(.4,1.4,.6,1) both;
  }
  @keyframes fadeInMsg {
    from { opacity: 0; transform: translateY(12px) scale(0.98); }
    to { opacity: 1; transform: none; }
  }
  .msg.user {
    background: linear-gradient(90deg, #ffe0b2 0%, #ffd699 100%);
    align-self: flex-end;
    text-align: right;
    color: #7a4c00;
    font-weight: 600;
    box-shadow: 0 2px 12px rgba(255,152,0,0.08);
  }
  .msg.assistant {
    background: linear-gradient(90deg, #e3f2fd 0%, #f0f7ff 100%);
    align-self: flex-start;
    text-align: left;
    color: #0d47a1;
    font-weight: 500;
    box-shadow: 0 2px 12px rgba(33,150,243,0.10);
  }
  .chat-input {
    display: flex;
    gap: 0.7em;
    margin-top: 0.7em;
    background: rgba(255,255,255,0.7);
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(33,150,243,0.04);
    padding: 0.4em 0.6em;
    border: 1.2px solid rgba(33,150,243,0.06);
  }
  .chat-input input {
    flex: 1;
    padding: 0.6em 1em;
    border-radius: 8px;
    border: none;
    font-size: 1.08em;
    background: transparent;
    outline: none;
    color: #222;
    font-family: inherit;
    transition: background 0.18s;
  }
  .chat-input input:focus {
    background: #f0f7ff;
  }
  .chat-input button {
    padding: 0.6em 1.4em;
    border-radius: 8px;
    border: none;
    background: linear-gradient(90deg, #ff9800 0%, #ff3e00 100%);
    color: #fff;
    font-weight: 700;
    font-size: 1.05em;
    cursor: pointer;
    box-shadow: 0 2px 8px rgba(255,152,0,0.10);
    transition: background 0.18s, transform 0.13s;
  }
  .chat-input button:disabled {
    background: #e0e0e0;
    color: #aaa;
    cursor: not-allowed;
    box-shadow: none;
  }
  .hint {
    margin-top: 1.2em;
    color: #6b7280;
    font-size: 1em;
    background: rgba(255,255,255,0.7);
    border-radius: 10px;
    padding: 1em 1.2em;
    box-shadow: 0 2px 8px rgba(33,150,243,0.04);
    border: 1px solid rgba(33,150,243,0.04);
    text-align: left;
    line-height: 1.6;
  }
  .hint code {
    background: #f0f7ff;
    color: #0d47a1;
    border-radius: 6px;
    padding: 0.2em 0.4em;
    font-size: 0.98em;
  }
  .map-view {
    width: 100%;
    height: 420px;
    border-radius: 18px;
    margin-bottom: 1.2em;
    box-shadow: 0 4px 24px rgba(33, 150, 243, 0.10);
    border: 1.5px solid rgba(33,150,243,0.08);
    background: rgba(240,245,255,0.7);
    overflow: hidden;
  }
  @media (max-width: 600px) {
    main {
      padding: 1.2em 0.2em 1.2em 0.2em;
    }
    .hero, .feature {
      padding: 1.2em 0.5em 1.2em 0.5em;
    }
    .map-view {
      height: 260px;
    }
    .chat-window {
      min-height: 120px;
      max-height: 180px;
    }
  }
</style>