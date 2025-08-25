import React, { useState, useEffect, useCallback, useRef } from 'react';
import {
  getFirestore,
  doc,
  setDoc,
  onSnapshot,
  query,
  orderBy,
  limit,
  collection
} from 'firebase/firestore';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  signInWithCustomToken,
  signInAnonymously,
  onAuthStateChanged,
} from 'firebase/auth';

const App = () => {
  // State for app navigation
  const [currentPage, setCurrentPage] = useState('home');
  const [showMenu, setShowMenu] = useState(false);

  // Universal state for errors, loading, and Firebase
  const [error, setError] = useState('');
  const [db, setDb] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [appId, setAppId] = useState('');
  const [uniqueName, setUniqueName] = useState('');

  // Text Prompt Generation state
  const [textPromptInput, setTextPromptInput] = useState('');
  const [generatedTextPrompts, setGeneratedTextPrompts] = useState([]);
  const [isTextLoading, setIsTextLoading] = useState(false);
  const [copiedIndex, setCopiedIndex] = useState(null);

  // Image to Prompt state
  const [imageFile, setImageFile] = useState(null);
  const [imagePromptResult, setImagePromptResult] = useState('');
  const [isImageProcessing, setIsImageProcessing] = useState(false);

  // Audio to Prompt state
  const [audioFile, setAudioFile] = useState(null);
  const [audioPromptResult, setAudioPromptResult] = useState('');
  const [isAudioProcessing, setIsAudioProcessing] = useState(false);
  const [isRecording, setIsRecording] = useState(false);
  const [audioChunks, setAudioChunks] = useState([]);
  const [mediaRecorder, setMediaRecorder] = useState(null);
  const audioFileInputRef = useRef(null);
  const imageFileInputRef = useRef(null);
  const imageCaptureInputRef = useRef(null);

  // Saved Prompts State
  const [savedPrompts, setSavedPrompts] = useState([]);
  const [isSavedPromptsLoading, setIsSavedPromptsLoading] = useState(false);

  // Constants for API endpoints
  const GEMINI_API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=";
  const API_KEY = ""; // Canvas environment provides this at runtime
  
  // Custom hook to detect if the user is on a mobile device
  const useIsMobile = () => {
    const [isMobile, setIsMobile] = useState(false);
    useEffect(() => {
      const userAgent = typeof navigator === 'undefined' ? '' : navigator.userAgent;
      const mobileCheck = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(userAgent);
      setIsMobile(mobileCheck);
    }, []);
    return isMobile;
  };
  const isMobile = useIsMobile();

  // Arrays for the unique name generator
  const adjectives = ['Sparkling', 'Cosmic', 'Witty', 'Galactic', 'Fluffy', 'Gigantic', 'Stealthy', 'Dancing', 'Mysterious', 'Vibrant', 'Curious', 'Zany', 'Electric', 'Fiery', 'Bouncing'];
  const nouns = ['Panda', 'Robot', 'Star', 'Whale', 'Noodle', 'Snail', 'Dragon', 'Pickle', 'Cactus', 'Unicorn', 'Meteor', 'Biscuit', 'Ninja', 'Squid', 'Taco'];

  // Deterministic function to generate a unique name from the user ID
  const generateUniqueName = (id) => {
    if (!id) return 'Guest User';
    const sum = id.split('').reduce((acc, char) => acc + char.charCodeAt(0), 0);
    const adjectiveIndex = sum % adjectives.length;
    const nounIndex = Math.floor(sum / adjectives.length) % nouns.length;
    return `${adjectives[adjectiveIndex]} ${nouns[nounIndex]}`;
  };

  // Initialize Firebase and handle authentication
  useEffect(() => {
    if (typeof __firebase_config !== 'undefined') {
      try {
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        setDb(firestore);
        const firebaseAuth = getAuth(app);

        const unsubscribe = onAuthStateChanged(firebaseAuth, (user) => {
          if (user) {
            setUserId(user.uid);
            setUniqueName(generateUniqueName(user.uid));
          } else {
            setUniqueName('Guest User');
          }
          setIsAuthReady(true);
        });

        if (typeof __initial_auth_token !== 'undefined') {
          signInWithCustomToken(firebaseAuth, __initial_auth_token)
            .catch(() => signInAnonymously(firebaseAuth));
        } else {
            signInAnonymously(firebaseAuth);
        }

        const currentAppId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        setAppId(currentAppId);

        return () => unsubscribe();
      } catch (e) {
        setError("Failed to initialize the application. Please try again.");
      }
    } else {
      setError("Firebase configuration is not available.");
    }
  }, []);

  // Use onSnapshot to fetch saved prompts in real-time
  useEffect(() => {
    if (db && userId && appId && currentPage === 'savedWork') {
      setIsSavedPromptsLoading(true);
      setError('');

      const q = query(
        collection(db, `artifacts/${appId}/users/${userId}/prompts`),
        orderBy("timestamp", "desc"),
        limit(10)
      );

      const unsubscribe = onSnapshot(q, (snapshot) => {
        const fetchedPrompts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setSavedPrompts(fetchedPrompts);
        setIsSavedPromptsLoading(false);
      }, (e) => {
        console.error("Failed to fetch saved prompts:", e);
        setError("Failed to load saved prompts. Please try again.");
        setIsSavedPromptsLoading(false);
      });

      return () => unsubscribe();
    }
  }, [db, userId, appId, currentPage]);

  // Utility function to convert file to base64
  const fileToBase64 = (file) => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result.split(',')[1]);
      reader.onerror = error => reject(error);
      reader.readAsDataURL(file);
    });
  };

  // Function to handle text prompt generation from Gemini
  const generateTextPrompts = useCallback(async () => {
    if (!textPromptInput.trim()) {
      setError('Please enter a prompt to get started.');
      return;
    }

    if (isTextLoading) return;

    setIsTextLoading(true);
    setError('');
    setGeneratedTextPrompts([]);

    const textPrompt = `Based on the following sentence, generate 5 distinct types of enhanced prompts that can be used with a large language model. Format the response as a JSON array of objects. Each object should have two keys: 'type' (string) and 'prompt' (string). The types should be:
    - Role-based: A prompt that assigns a specific role or persona to the AI.
    - Step-by-step: A prompt that breaks down the task into sequential steps.
    - Constraint-based: A prompt that adds specific rules, limits, or constraints.
    - Conversational: A prompt that frames the request as a natural, conversational exchange.
    - Creative/Storytelling: A prompt that encourages a creative or narrative response.
    
    Original Sentence: "${textPromptInput}"`;

    const payload = {
      contents: [{ role: "user", parts: [{ text: textPrompt }] }],
      generationConfig: {
        responseMimeType: "application/json",
        responseSchema: {
          type: "ARRAY",
          items: {
            type: "OBJECT",
            properties: { "type": { type: "STRING" }, "prompt": { type: "STRING" } },
            propertyOrdering: ["type", "prompt"]
          }
        }
      }
    };

    let retries = 0;
    const maxRetries = 5;

    const executeFetch = async () => {
      try {
        const response = await fetch(`${GEMINI_API_URL}${API_KEY}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error(`API response was not ok: ${response.status}`);

        const result = await response.json();
        const jsonText = result?.candidates?.[0]?.content?.parts?.[0]?.text;
        const parsedJson = JSON.parse(jsonText);
        setGeneratedTextPrompts(parsedJson);

        if (db && userId && appId) {
          const docRef = doc(db, `artifacts/${appId}/users/${userId}/prompts`, `${Date.now()}`);
          await setDoc(docRef, {
            original: textPromptInput,
            generated: parsedJson,
            timestamp: new Date(),
            type: 'text'
          });
        }
      } catch (e) {
        console.error("API call error:", e);
        if (retries < maxRetries) {
          retries++;
          const delay = Math.pow(2, retries) * 1000;
          setTimeout(executeFetch, delay);
        } else {
          setError(`Failed to generate prompts. Error: ${e.message}`);
        }
      } finally {
        setIsTextLoading(false);
      }
    };
    executeFetch();
  }, [textPromptInput, isTextLoading, db, userId, appId]);

  // Function to handle image to prompt generation
  const generatePromptFromImage = useCallback(async () => {
    if (!imageFile) {
      setError('Please upload an image file or take a picture.');
      return;
    }

    setIsImageProcessing(true);
    setError('');
    setImagePromptResult('');

    try {
      const base64Data = await fileToBase64(imageFile);
      const prompt = "Analyze this image and provide a detailed and creative prompt that could be used to generate a similar image or text description.";

      const payload = {
        contents: [{
          role: "user",
          parts: [
            { text: prompt },
            {
              inlineData: {
                mimeType: imageFile.type,
                data: base64Data
              }
            }
          ]
        }]
      };

      const response = await fetch(`${GEMINI_API_URL}${API_KEY}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!response.ok) throw new Error(`API response was not ok: ${response.status}`);
      
      const result = await response.json();
      const generatedText = result?.candidates?.[0]?.content?.parts?.[0]?.text;
      setImagePromptResult(generatedText);

      if (db && userId && appId) {
        const docRef = doc(db, `artifacts/${appId}/users/${userId}/prompts`, `${Date.now()}`);
        await setDoc(docRef, {
          original: `Prompt from image: ${imageFile.name}`,
          generated: [{ type: "Image-based Prompt", prompt: generatedText }],
          timestamp: new Date(),
          type: 'image',
          image: URL.createObjectURL(imageFile) // Saving the image data URL for display
        });
      }

    } catch (e) {
      setError(`Failed to process image. Error: ${e.message}`);
    } finally {
      setIsImageProcessing(false);
    }
  }, [imageFile]);

  // Function to handle audio to prompt generation
  const generatePromptFromAudio = useCallback(async () => {
    if (!audioFile) {
      setError('Please upload an audio file or record audio.');
      return;
    }

    setIsAudioProcessing(true);
    setError('');
    setAudioPromptResult('');

    try {
      const base64Data = await fileToBase64(audioFile);
      const prompt = "Transcribe the audio and then create a detailed, enhanced prompt from the transcription. The prompt should be suitable for a generative AI model.";
      
      const payload = {
        contents: [{
          role: "user",
          parts: [
            { text: prompt },
            {
              inlineData: {
                mimeType: audioFile.type,
                data: base64Data
              }
            }
          ]
        }]
      };

      const response = await fetch(`${GEMINI_API_URL}${API_KEY}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!response.ok) throw new Error(`API response was not ok: ${response.status}`);
      
      const result = await response.json();
      const generatedText = result?.candidates?.[0]?.content?.parts?.[0]?.text;
      setAudioPromptResult(generatedText);
      
      if (db && userId && appId) {
        const docRef = doc(db, `artifacts/${appId}/users/${userId}/prompts`, `${Date.now()}`);
        await setDoc(docRef, {
          original: `Prompt from audio: ${audioFile.name}`,
          generated: [{ type: "Audio-based Prompt", prompt: generatedText }],
          timestamp: new Date(),
          type: 'audio',
          audio: URL.createObjectURL(audioFile) // Saving the audio data URL for display
        });
      }

    } catch (e) {
      setError(`Failed to process audio. Error: ${e.message}`);
    } finally {
      setIsAudioProcessing(false);
    }
  }, [audioFile]);

  // Function to copy text to the clipboard
  const handleCopy = (text) => {
    const el = document.createElement('textarea');
    el.value = text;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    alert('Prompt copied to clipboard!');
  };

  // Recording functionality for audio
  const startRecording = async () => {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const newMediaRecorder = new MediaRecorder(stream);
        newMediaRecorder.ondataavailable = (e) => {
            setAudioChunks((prev) => [...prev, e.data]);
        };
        newMediaRecorder.onstop = () => {
            const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
            setAudioFile(audioBlob);
            setAudioChunks([]);
        };
        setMediaRecorder(newMediaRecorder);
        newMediaRecorder.start();
        setIsRecording(true);
        setError('');
    } catch (e) {
        setError("Error accessing microphone. Please ensure permissions are granted.");
    }
  };

  const stopRecording = () => {
    if (mediaRecorder && mediaRecorder.state === 'recording') {
        mediaRecorder.stop();
        setIsRecording(false);
    }
  };
  
  if (!isAuthReady) {
    return (
      <div className="flex justify-center items-center min-h-screen bg-[#191970] text-[#FFD700]">
        <div className="text-center">
          <svg className="animate-spin h-8 w-8 text-[#FFD700] mx-auto" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          <p className="mt-4 text-lg text-gray-300">Loading application...</p>
        </div>
      </div>
    );
  }

  // A simple router to show different pages
  const renderPage = () => {
    switch (currentPage) {
      case 'home':
        return (
          <div className="flex flex-col items-center justify-center space-y-8 mt-12 text-center">
            <h1 className="text-4xl sm:text-5xl lg:text-6xl font-extrabold text-[#FFD700] leading-tight">
              Promptify AI
            </h1>
            <p className="text-gray-300 text-lg max-w-2xl">
              Promptify AI is a creative nexus where ideas take flight. We empower you to transform raw inspiration—from text, images, and sound—into meticulously crafted prompts for cutting-edge generative AI. Unlock the full potential of your creative vision.
            </p>
            <div className="flex flex-col sm:flex-row justify-center space-y-4 sm:space-y-0 sm:space-x-8">
              <button 
                onClick={() => {setCurrentPage('text'); setShowMenu(false);}}
                className="w-full sm:w-64 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#FFD700] hover:bg-yellow-500 active:scale-95"
              >
                Enhance Text Prompt
              </button>
              <button 
                onClick={() => {setCurrentPage('image'); setShowMenu(false);}}
                className="w-full sm:w-64 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#FFD700] hover:bg-yellow-500 active:scale-95"
              >
                Image to Prompt
              </button>
              <button 
                onClick={() => {setCurrentPage('audio'); setShowMenu(false);}}
                className="w-full sm:w-64 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#FFD700] hover:bg-yellow-500 active:scale-95"
              >
                Audio to Prompt
              </button>
            </div>
          </div>
        );
      case 'text':
        return (
          <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
            <div className="flex justify-between items-center">
              <h2 className="text-3xl font-bold text-[#FFD700]">Enhance Text Prompt</h2>
              <button 
                onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                className="text-[#87CEEB] hover:text-[#FFD700]"
              >
                ← Back to Home
              </button>
            </div>
            <p className="text-gray-300 text-md">
              Refine your creative ideas. Our powerful AI takes your simple sentences and transforms them into a diverse set of five unique, enhanced prompts. Explore different narrative angles, constraints, and personas to get precisely the response you're looking for.
            </p>
            <textarea
              id="text-prompt-input"
              rows="4"
              value={textPromptInput}
              onChange={(e) => setTextPromptInput(e.target.value)}
              className="w-full p-4 bg-[#0F0F3D] text-[#FFD700] rounded-xl focus:outline-none focus:ring-2 focus:ring-[#87CEEB] transition-shadow duration-300 resize-none"
              placeholder="Enter a simple sentence to generate enhanced prompts..."
            ></textarea>
            <button
              onClick={() => { generateTextPrompts(); setShowMenu(false); }}
              disabled={isTextLoading}
              className={`w-full px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#FFD700] hover:bg-yellow-500 active:scale-95 ${
                isTextLoading ? 'opacity-50 cursor-not-allowed' : ''
              }`}
            >
              {isTextLoading ? (
                <svg className="animate-spin h-6 w-6 text-white inline-block" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                  <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                  <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
              ) : 'Generate Text Prompts'}
            </button>
            {generatedTextPrompts.length > 0 && (
              <div className="grid grid-cols-1 md:grid-cols-2 gap-8 mt-8">
                {generatedTextPrompts.map((p, index) => (
                  <div
                    key={index}
                    className="bg-[#0F0F3D] p-6 rounded-3xl shadow-inner space-y-4 border-l-4 border-[#87CEEB] transition-shadow duration-300 hover:shadow-xl"
                  >
                    <div className="flex items-center space-x-2 justify-between">
                      <span className="text-[#FFD700] font-bold text-lg">{p.type}</span>
                      <button
                        onClick={() => handleCopy(p.prompt, index)}
                        className="text-sm font-semibold py-1 px-3 rounded-full transition-transform transform bg-[#FFD700] text-[#191970] hover:scale-105 active:scale-95"
                      >
                        {copiedIndex === index ? 'Copied!' : 'Copy'}
                      </button>
                    </div>
                    <p className="text-gray-300 leading-relaxed break-words whitespace-pre-wrap">
                      {p.prompt}
                    </p>
                  </div>
                ))}
              </div>
            )}
          </div>
        );
      case 'image':
        return (
          <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
            <div className="flex justify-between items-center">
              <h2 className="text-3xl font-bold text-[#FFD700]">Image to Prompt</h2>
              <button 
                onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                className="text-[#87CEEB] hover:text-[#FFD700]"
              >
                ← Back to Home
              </button>
            </div>
            <p className="text-gray-300 text-md">
              Turn visual inspiration into text. Upload an image or capture a new one with your camera. Our system will intelligently analyze its contents and generate a detailed, descriptive prompt.
            </p>
            <div className="bg-[#0F0F3D] p-6 rounded-xl space-y-4">
              <p className="text-sm text-gray-400">
                <span className="font-bold">Note:</span> For best results, please upload JPG images. The "Take a Picture" option is optimized for mobile devices.
              </p>
              <div className="flex flex-col sm:flex-row gap-4">
                <input 
                  ref={imageFileInputRef}
                  type="file" 
                  onChange={(e) => setImageFile(e.target.files[0])} 
                  accept="image/jpeg" 
                  className="hidden" 
                />
                <button
                  onClick={() => { imageFileInputRef.current.click(); setShowMenu(false); }}
                  className="flex-1 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#4169E1] hover:bg-blue-700 active:scale-95"
                >
                  Click to Upload File
                </button>
                {isMobile && (
                  <>
                    <input 
                      ref={imageCaptureInputRef}
                      type="file" 
                      onChange={(e) => setImageFile(e.target.files[0])} 
                      accept="image/jpeg" 
                      capture="camera"
                      className="hidden" 
                    />
                    <button
                      onClick={() => { imageCaptureInputRef.current.click(); setShowMenu(false); }}
                      className="flex-1 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#4169E1] hover:bg-blue-700 active:scale-95"
                    >
                      Take a Picture
                    </button>
                  </>
                )}
              </div>
              {imageFile && (
                <div className="mt-4 text-center text-gray-300">
                  <p>Selected file: {imageFile.name}</p>
                  <img 
                    src={URL.createObjectURL(imageFile)} 
                    alt="Preview" 
                    className="mt-2 rounded-lg mx-auto max-h-64 shadow-md"
                  />
                </div>
              )}
            </div>
            <button
                onClick={() => { generatePromptFromImage(); setShowMenu(false); }}
                disabled={isImageProcessing || !imageFile}
                className={`w-full px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#FFD700] hover:bg-yellow-500 active:scale-95 ${
                    isImageProcessing || !imageFile ? 'opacity-50 cursor-not-allowed' : ''
                }`}
            >
                {isImageProcessing ? (
                  <svg className="animate-spin h-6 w-6 text-white inline-block" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                ) : 'Generate Prompt'}
            </button>
            {imagePromptResult && (
              <div className="bg-[#0F0F3D] p-6 rounded-3xl shadow-inner space-y-4 border-l-4 border-[#FFD700] mt-8">
                <span className="text-[#FFD700] font-bold text-lg">Generated Prompt:</span>
                <p className="text-gray-300 leading-relaxed break-words whitespace-pre-wrap">
                  {imagePromptResult}
                </p>
              </div>
            )}
          </div>
        );
      case 'audio':
        return (
          <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
            <div className="flex justify-between items-center">
              <h2 className="text-3xl font-bold text-[#FFD700]">Audio to Prompt</h2>
              <button 
                onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                className="text-[#87CEEB] hover:text-[#FFD700]"
              >
                ← Back to Home
              </button>
            </div>
            <p className="text-gray-300 text-md">
              Give a voice to your prompts. Record your thoughts or upload an audio file, and our AI will transcribe the content into an elegant, enhanced prompt.
            </p>
            <div className="bg-[#0F0F3D] p-6 rounded-xl space-y-4">
              <div className="flex flex-col sm:flex-row gap-4">
                <input 
                  ref={audioFileInputRef}
                  type="file" 
                  onChange={(e) => setAudioFile(e.target.files[0])} 
                  accept="audio/*" 
                  className="hidden" 
                />
                <button
                  onClick={() => { audioFileInputRef.current.click(); setShowMenu(false); }}
                  className="flex-1 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white bg-[#4169E1] hover:bg-blue-700 active:scale-95"
                >
                  Select Audio File
                </button>
                <button
                  onClick={() => { isRecording ? stopRecording() : startRecording(); setShowMenu(false); }}
                  className={`flex-1 px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-white ${
                    isRecording ? 'bg-red-500 hover:bg-red-600' : 'bg-[#32CD32] hover:bg-green-700'
                  }`}
                >
                  {isRecording ? 'Stop Recording' : 'Start Recording'}
                </button>
              </div>
              {audioFile && (
                <div className="mt-4 text-center text-gray-300">
                  <p>Selected file: {audioFile.name}</p>
                  <audio controls src={URL.createObjectURL(audioFile)} className="w-full mt-2" />
                </div>
              )}
            </div>
            <button
                onClick={() => { generatePromptFromAudio(); setShowMenu(false); }}
                disabled={isAudioProcessing || !audioFile}
                className={`w-full px-8 py-4 font-bold rounded-xl shadow-lg transition-transform transform text-[#191970] ${
                    isAudioProcessing || !audioFile ? 'bg-[#FFD700] cursor-not-allowed' : 'bg-[#FFD700] hover:bg-yellow-500 active:scale-95'
                }`}
            >
                {isAudioProcessing ? (
                  <svg className="animate-spin h-6 w-6 text-white inline-block" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                ) : 'Generate Prompt'}
            </button>
            {audioPromptResult && (
              <div className="bg-[#0F0F3D] p-6 rounded-3xl shadow-inner space-y-4 border-l-4 border-[#FFD700] mt-8">
                <span className="text-[#FFD700] font-bold text-lg">Generated Prompt:</span>
                <p className="text-gray-300 leading-relaxed break-words whitespace-pre-wrap">
                  {audioPromptResult}
                </p>
              </div>
            )}
          </div>
        );
      case 'savedWork':
        return (
          <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
            <div className="flex justify-between items-center">
              <h2 className="text-3xl font-bold text-[#FFD700]">Saved Work</h2>
              <button 
                onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                className="text-[#87CEEB] hover:text-[#FFD700]"
              >
                ← Back to Home
              </button>
            </div>
            <p className="text-gray-300 text-md">
              Your last 10 enhanced text prompts are saved here.
            </p>
            {isSavedPromptsLoading && (
              <div className="flex justify-center items-center">
                <svg className="animate-spin h-8 w-8 text-[#87CEEB] mx-auto" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                  <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                  <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
              </div>
            )}
            {!isSavedPromptsLoading && savedPrompts.length === 0 && (
              <div className="text-center text-gray-500 p-8">
                No saved prompts found. Generate some prompts to see them here!
              </div>
            )}
            {!isSavedPromptsLoading && savedPrompts.length > 0 && (
              <div className="grid grid-cols-1 gap-8">
                {savedPrompts.map((savedDoc) => (
                  <div key={savedDoc.id} className="bg-[#0F0F3D] p-6 rounded-3xl shadow-inner space-y-4">
                    <p className="text-gray-300 text-sm">
                      <span className="font-bold text-[#FFD700]">Original:</span> {savedDoc.original}
                    </p>
                    <div className="space-y-4">
                      {savedDoc.generated.map((p, index) => (
                        <div key={index} className="bg-[#191970] p-4 rounded-xl shadow-sm border-l-2 border-[#87CEEB]">
                          <div className="flex justify-between items-center mb-2">
                            <span className="font-bold text-sm text-[#FFD700]">{p.type}</span>
                            <button
                              onClick={() => handleCopy(p.prompt, index)}
                              className="text-xs font-semibold py-1 px-2 rounded-full transition-transform transform bg-[#FFD700] text-[#191970] hover:scale-105 active:scale-95"
                            >
                              Copy
                            </button>
                          </div>
                          <p className="text-gray-300 text-sm">{p.prompt}</p>
                        </div>
                      ))}
                    </div>
                  </div>
                ))}
              </div>
            )}
          </div>
        );
        case 'about':
        return (
          <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
            <div className="flex justify-between items-center">
              <h2 className="text-3xl font-bold text-[#FFD700]">About Promptify AI</h2>
              <button 
                onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                className="text-[#87CEEB] hover:text-[#FFD700]"
              >
                ← Back to Home
              </button>
            </div>
            <p className="text-gray-300 text-md leading-relaxed">
              **Promptify AI** is designed to democratize the power of generative AI. Many people struggle to get the precise, high-quality results they want from AI models because they don't know how to craft effective prompts. Our mission is to bridge that gap by offering a set of intuitive tools that transform simple ideas into powerful, detailed, and enhanced prompts.
            </p>
            <h3 className="text-2xl font-bold text-[#FFD700] mt-6">Our Features</h3>
            <ul className="list-disc list-inside space-y-2 text-gray-300 leading-relaxed">
              <li>
                **Enhance Text Prompt**: Our flagship feature takes a simple sentence and expands it into five distinct, professionally structured prompts. These prompts apply different techniques, such as defining a persona (Role-based), breaking down a task (Step-by-step), or adding creative flair (Creative/Storytelling), giving you a variety of powerful starting points.
              </li>
              <li>
                **Image to Prompt**: Get inspired by visuals. Upload an image or capture one with your camera, and our AI will analyze its content to generate a rich, descriptive prompt. This is perfect for artists, designers, or anyone who wants to translate a visual concept into a textual description for another AI model.
              </li>
              <li>
                **Audio to Prompt**: Capture ideas on the go. Simply record a spoken thought or upload an audio file. Our system will transcribe the audio and intelligently craft it into an enhanced prompt, ensuring your creative moments are never lost.
              </li>
              <li>
                **Saved Work**: Never lose a great idea. We save your last 10 enhanced text prompts so you can easily revisit and reuse them whenever inspiration strikes.
              </li>
            </ul>
            <p className="text-gray-300 text-md leading-relaxed">
              We believe that the future of creativity is collaborative, and Promptify AI is here to make your collaboration with AI smarter, more effective, and more enjoyable.
            </p>
          </div>
        );
        case 'contact':
        return (
            <div className="w-full max-w-4xl mx-auto space-y-8 bg-[#191970] p-8 rounded-2xl shadow-xl">
              <div className="flex justify-between items-center">
                <h2 className="text-3xl font-bold text-[#FFD700]">Contact</h2>
                <button 
                  onClick={() => {setCurrentPage('home'); setShowMenu(false);}}
                  className="text-[#87CEEB] hover:text-[#FFD700]"
                >
                  ← Back to Home
                </button>
              </div>
              <p className="text-gray-300 text-md leading-relaxed">
                Should you wish to communicate directly with the architect of this domain, you may use the contact information provided below. Your inquiries are welcome and will be received with due regard.
              </p>
              <div className="bg-[#0F0F3D] p-6 rounded-2xl border-l-4 border-[#87CEEB] shadow-sm space-y-4">
                <div>
                  <h3 className="text-xl font-bold text-[#FFD700]">Creator's Details</h3>
                  <div className="mt-2 space-y-1 text-gray-300">
                    <p className="font-semibold">Creator: <span className="font-normal text-[#87CEEB]">MOHAN GOPI</span></p>
                    <p className="font-semibold">Mail ID: <a href="mailto:promptifyaibyvv@gmail.com" className="text-[#87CEEB] hover:underline">promptifyaibyvv@gmail.com</a></p>
                  </div>
                </div>
              </div>
            </div>
        );
      default:
        return null;
    }
  };

  return (
    <div className="min-h-screen font-sans antialiased" style={{
      background: 'radial-gradient(circle, #252F5A 0%, #1A1A2E 100%)'
    }}>
      <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;700&display=swap" />

      {/* Header/Nav Bar */}
      <nav className="flex items-center justify-between p-4 md:px-8 shadow-sm bg-black bg-opacity-30">
        <div className="flex items-center space-x-2">
          <span className="font-bold text-xl text-[#FFD700]">Promptify AI</span>
        </div>
        
        <div className="relative">
          <button onClick={() => setShowMenu(!showMenu)} className="text-white p-2 rounded-lg hover:bg-gray-700 transition-colors duration-300">
            <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 6h16M4 12h16m-7 6h7" />
            </svg>
          </button>
          {showMenu && (
            <div className="absolute right-0 mt-2 w-48 bg-gray-800 rounded-lg shadow-xl py-2 z-50">
              <div className="px-4 py-2 text-white font-bold border-b border-gray-700">
                Info
              </div>
              <button onClick={() => { setCurrentPage('about'); setShowMenu(false); }} className="block w-full text-left px-4 py-2 text-gray-300 hover:bg-gray-700">About</button>
              <button onClick={() => { setCurrentPage('savedWork'); setShowMenu(false); }} className="block w-full text-left px-4 py-2 text-gray-300 hover:bg-gray-700">Saved Work</button>
              <button onClick={() => { setCurrentPage('contact'); setShowMenu(false); }} className="block w-full text-left px-4 py-2 text-gray-300 hover:bg-gray-700">Contact</button>
              <div className="px-4 py-2 text-white font-bold border-t border-gray-700 mt-2">
                User
              </div>
              <div className="px-4 py-2 text-gray-400 text-sm">
                <span className="font-bold text-yellow-400">Name:</span> {uniqueName || 'N/A'}
              </div>
              <div className="px-4 py-2 text-gray-400 text-xs break-all">
                <span className="font-bold text-yellow-400">ID:</span> {userId || 'N/A'}
              </div>
            </div>
          )}
        </div>
      </nav>

      <main className="container mx-auto p-8">
        {renderPage()}
      </main>

      {error && (
        <div className="fixed bottom-8 left-1/2 transform -translate-x-1/2 bg-red-800 p-4 rounded-xl text-red-100 text-center z-50">
          <p>{error}</p>
        </div>
      )}
    </div>
  );
};

export default App;
