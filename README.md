<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0" />
  <title>Maestro Sergio - Estudio de Arte Digital</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body, #root { height: 100%; width: 100%; overflow: hidden; }
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background: #0c0b09;
      color: #d8cfc0;
      line-height: 1.6;
    }
    input::placeholder { color: #888; }
    /* Scrollbar personalizada */
    ::-webkit-scrollbar { width: 6px; }
    ::-webkit-scrollbar-track { background: #0c0b09; }
    ::-webkit-scrollbar-thumb { background: #2a2520; border-radius: 10px; }
  </style>
</head>
<body>
  <div id="root"></div>

  <script type="text/babel">
    const { useState, useRef, useEffect } = React;

    // API KEY de Google Gemini
    const MI_LLAVE_GEMINI = "AIzaSyBTNo-dfaLJ056uz7Gk0e8UCY6YHNeiMao";

    const LEVELS = ["Principiante", "Intermedio", "Avanzado"];

    function ArteApp() {
      const [screen, setScreen] = useState("home");
      const [messages, setMessages] = useState([]);
      const [input, setInput] = useState("");
      const [loading, setLoading] = useState(false);
      const [level, setLevel] = useState("Intermedio");
      const [pendingImage, setPendingImage] = useState(null);
      const endRef = useRef(null);

      // Auto-scroll al final del chat
      useEffect(() => { 
        endRef.current?.scrollIntoView({ behavior: "smooth" }); 
      }, [messages, loading]);

      const handleFileChange = (e) => {
        const file = e.target.files?.[0];
        if (!file) return;
        
        const reader = new FileReader();
        reader.onload = (ev) => {
          setPendingImage({ 
            file, 
            preview: ev.target.result,
            base64: ev.target.result.split(',')[1],
            mimeType: file.type 
          });
        };
        reader.readAsDataURL(file);
        // Reset del input file para poder subir la misma imagen si se borra
        e.target.value = "";
      };

      const sendMessage = async () => {
        const userText = input.trim();
        if (!userText && !pendingImage) return;

        const currentImage = pendingImage; 
        setLoading(true); 
        setInput(""); 
        setPendingImage(null);
        if(screen !== "chat") setScreen("chat");

        // Añadir mensaje del usuario a la lista
        const userMsg = { 
          role: "user", 
          content: userText || "Analiza mi obra", 
          userImagePreview: currentImage?.preview 
        };
        setMessages(prev => [...prev, userMsg]);

        try {
          // System Prompt para darle personalidad al modelo
          const systemPrompt = `Actúa como Sergio, un maestro de dibujo y pintura experto. 
          Tu objetivo es dar feedback técnico, correcciones anatómicas, de color o composición. 
          Nivel del alumno: ${level}. Sé directo pero inspirador.`;
          
          let parts = [{ text: `${systemPrompt}\n\nAlumno dice: ${userText || "Por favor, analiza mi imagen."}` }];
          
          if (currentImage) {
            parts.push({
              inline_data: {
                mime_type: currentImage.mimeType,
                data: currentImage.base64
              }
            });
          }

          const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${MI_LLAVE_GEMINI}`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ contents: [{ parts }] })
          });

          const data = await response.json();
          
          if (data.error) throw new Error(data.error.message);

          const responseText = data.candidates[0].content.parts[0].text;
          setMessages(prev => [...prev, { role: "assistant", content: responseText }]);
        } catch (e) {
          console.error("Error API:", e);
          setMessages(prev => [...prev, { role: "assistant", content: "Lo siento, hubo un problema técnico: " + e.message }]);
        } finally {
          setLoading(false);
        }
      };

      // --- PANTALLA DE INICIO ---
      if (screen === "home") return (
        <div style={{ height:"100vh", background:"#0f0e0c", display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", padding:24, textAlign:"center" }}>
          <div style={{ border: "1px solid #c97d3a", padding: "40px", borderRadius: "20px" }}>
            <h1 style={{ color:"#e8c87a", fontSize: "2.5rem", marginBottom:10 }}>Maestro Sergio</h1>
            <p style={{ color:"#6a5a4a", fontSize:12, letterSpacing:4, marginBottom:40, textTransform: "uppercase" }}>Estudio de Arte Digital</p>
            
            <div style={{ marginBottom:30 }}>
              <p style={{ marginBottom:15, fontSize:14, color: "#d8cfc0" }}>¿Cuál es tu nivel actual?</p>
              <div style={{ display: "flex", justifyContent: "center", gap: 10 }}>
                {LEVELS.map(l => (
                  <button key={l} onClick={()=>setLevel(l)} 
                    style={{ 
                      padding:"12px 20px", 
                      background:level===l?"#c97d3a":"#1e1b17", 
                      color:"#fff", 
                      border: "1px solid",
                      borderColor: level===l?"#fff":"#333", 
                      borderRadius:8, 
                      cursor:"pointer",
                      transition: "0.3s"
                    }}>
                    {l}
                  </button>
                ))}
              </div>
            </div>
            
            <button onClick={() => setScreen("chat")} 
              style={{ width:"100%", maxWidth:300, padding:18, background:"#c97d3a", color:"#fff", border:"none", borderRadius:12, fontSize:18, fontWeight:"bold", cursor:"pointer", boxShadow: "0 4px 15px rgba(201, 125, 58, 0.3)" }}>
              Entrar al Estudio
            </button>
          </div>
        </div>
      );

      // --- PANTALLA DE CHAT ---
      return (
        <div style={{ height:"100vh", display:"flex", flexDirection:"column", background:"#0c0b09" }}>
          {/* Cabecera */}
          <header style={{ padding:"15px 20px", background:"#14120f", borderBottom:"1px solid #2a2520", display:"flex", alignItems:"center", justifyContent: "space-between" }}>
            <div style={{ display: "flex", alignItems: "center" }}>
              <button onClick={()=>setScreen("home")} style={{ background:"none", border:"none", color:"#c97d3a", marginRight:15, cursor:"pointer", fontSize: 18 }}>✕</button>
              <span style={{ color:"#e8c87a", fontWeight: "bold" }}>Maestro Sergio <span style={{ fontWeight: "normal", color: "#6a5a4a", marginLeft: 10 }}>| {level}</span></span>
            </div>
          </header>

          {/* Área de Chat */}
          <main style={{ flex:1, overflowY:"auto", padding:"20px", display:"flex", flexDirection:"column", gap:20 }}>
            {messages.length === 0 && (
              <div style={{ textAlign: "center", marginTop: 50, color: "#444" }}>
                <p>Bienvenido al estudio. Adjunta una obra o haz una pregunta técnica.</p>
              </div>
            )}
            
            {messages.map((msg, i) => (
              <div key={i} style={{ alignSelf: msg.role==="user"?"flex-end":"flex-start", maxWidth:"85%" }}>
                {msg.userImagePreview && (
                  <img src={msg.userImagePreview} style={{ width:"100%", maxWidth: 300, borderRadius:12, marginBottom:8, border:"2px solid #c97d3a" }} />
                )}
                <div style={{ 
                  padding:14, 
                  borderRadius:15, 
                  background:msg.role==="user"?"#c97d3a":"#1e1b17", 
                  color:"#fff", 
                  fontSize:15,
                  boxShadow: "0 2px 10px rgba(0,0,0,0.3)",
                  whiteSpace: "pre-wrap"
                }}>
                  {msg.content}
                </div>
              </div>
            ))}
            
            {loading && (
              <div style={{ color:"#c97d3a", fontSize:14, fontStyle: "italic", marginLeft: 10 }}>
                Sergio está observando...
              </div>
            )}
            <div ref={endRef} />
          </main>

          {/* Área de Entrada */}
          <footer style={{ padding:20, background:"#14120f", borderTop:"1px solid #2a2520" }}>
            {pendingImage && (
              <div style={{ marginBottom:15, position:"relative", display:"inline-block" }}>
                <img src={pendingImage.preview} style={{ height:80, borderRadius:8, border:"2px solid #c97d3a" }} />
                <button onClick={()=>setPendingImage(null)} 
                  style={{ position:"absolute", top:-8, right:-8, background:"#ff4444", color:"white", border:"none", borderRadius:"50%", width:24, height:24, cursor:"pointer", fontWeight: "bold" }}>
                  ✕
                </button>
              </div>
            )}
            
            <div style={{ display:"flex", gap:12 }}>
              <input type="file" id="file" hidden accept="image/*" onChange={handleFileChange} />
              <button onClick={()=>document.getElementById('file').click()} 
                title="Adjuntar dibujo"
                style={{ background:"#1e1b17", border:"1px solid #333", color:"#c97d3a", padding:"0 18px", borderRadius:10, cursor:"pointer", fontSize: 20 }}>
                🎨
              </button>
              
              <input 
                value={input} 
                onChange={e=>setInput(e.target.value)} 
                onKeyDown={e=>e.key==="Enter" && sendMessage()} 
                placeholder="Escribe tu consulta aquí..." 
                style={{ 
                  flex:1, 
                  padding:"12px 15px", 
                  borderRadius:10, 
                  border:"1px solid #333", 
                  background:"#1e1b17", 
                  color:"#fff",
                  fontSize: 16
                }} 
              />
              
              <button onClick={sendMessage} disabled={loading}
                style={{ 
                  background:loading?"#444":"#c97d3a", 
                  color:"#fff", 
                  border:"none", 
                  padding:"0 25px", 
                  borderRadius:10, 
                  cursor:"pointer", 
                  fontWeight:"bold",
                  transition: "0.2s"
                }}>
                ENVIAR
              </button>
            </div>
          </footer>
        </div>
      );
    }

    const root = ReactDOM.createRoot(document.getElementById("root"));
    root.render(<ArteApp />);
  </script>
</body>
</html>

