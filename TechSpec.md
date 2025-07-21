# Dewey - Technical Implementation Guide
## Gemini 2.5 Flash Live API Integration

---

## 1. Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌────────────────────┐
│   React Native  │    │  Supabase Edge   │    │   Gemini Live API  │
│                 │    │    Functions     │    │                    │
│ ┌─────────────┐ │    │                  │    │ ┌────────────────┐ │
│ │ Voice Input │ │───▶│ WebSocket Proxy  │───▶│ │ Audio Processing│ │
│ └─────────────┘ │    │                  │    │ └────────────────┘ │
│ ┌─────────────┐ │    │                  │    │ ┌────────────────┐ │
│ │Audio Output │ │◀───│ Real-time Stream │◀───│ │ Voice Response │ │
│ └─────────────┘ │    │                  │    │ └────────────────┘ │
└─────────────────┘    └──────────────────┘    └────────────────────┘
```

## 2. React Native Audio Implementation

### 2.1 Audio Recording Component

```typescript
// components/DeweyMicrophone.tsx
import React, { useState, useRef, useEffect } from 'react';
import { View, TouchableOpacity, Animated } from 'react-native';
import Voice from '@react-native-voice/voice';
import { Mic, MicOff } from 'lucide-react-native';

interface DeweyMicrophoneProps {
  onAudioData: (audioData: Uint8Array) => void;
  onRecordingStart: () => void;
  onRecordingEnd: () => void;
}

export const DeweyMicrophone: React.FC<DeweyMicrophoneProps> = ({
  onAudioData,
  onRecordingStart,
  onRecordingEnd
}) => {
  const [isRecording, setIsRecording] = useState(false);
  const [isProcessing, setIsProcessing] = useState(false);
  const pulseAnimation = useRef(new Animated.Value(1)).current;
  const audioBufferRef = useRef<number[]>([]);

  useEffect(() => {
    // Configure voice recognition for audio capture
    Voice.onSpeechStart = handleSpeechStart;
    Voice.onSpeechEnd = handleSpeechEnd;
    Voice.onSpeechResults = handleSpeechResults;
    Voice.onSpeechError = handleSpeechError;

    return () => {
      Voice.destroy();
    };
  }, []);

  const handleSpeechStart = () => {
    console.log('Speech started');
    startPulseAnimation();
  };

  const handleSpeechEnd = () => {
    console.log('Speech ended');
    stopPulseAnimation();
    processAudioBuffer();
  };

  const handleSpeechResults = (event: any) => {
    // We're not using speech-to-text results, 
    // but we need raw audio data for Gemini
    console.log('Speech results:', event.value);
  };

  const handleSpeechError = (event: any) => {
    console.error('Speech error:', event.error);
    setIsRecording(false);
    setIsProcessing(false);
  };

  const startRecording = async () => {
    try {
      setIsRecording(true);
      onRecordingStart();
      audioBufferRef.current = [];
      
      // Start voice recognition (this will capture audio)
      await Voice.start('en-US');
    } catch (error) {
      console.error('Error starting recording:', error);
      setIsRecording(false);
    }
  };

  const stopRecording = async () => {
    try {
      setIsRecording(false);
      setIsProcessing(true);
      await Voice.stop();
      onRecordingEnd();
    } catch (error) {
      console.error('Error stopping recording:', error);
    }
  };

  const processAudioBuffer = () => {
    if (audioBufferRef.current.length > 0) {
      // Convert to PCM format required by Gemini (16kHz, 16-bit)
      const audioData = new Uint8Array(audioBufferRef.current);
      onAudioData(audioData);
    }
    setIsProcessing(false);
  };

  const startPulseAnimation = () => {
    Animated.loop(
      Animated.sequence([
        Animated.timing(pulseAnimation, {
          toValue: 1.2,
          duration: 600,
          useNativeDriver: true,
        }),
        Animated.timing(pulseAnimation, {
          toValue: 1,
          duration: 600,
          useNativeDriver: true,
        }),
      ])
    ).start();
  };

  const stopPulseAnimation = () => {
    pulseAnimation.stopAnimation();
    Animated.timing(pulseAnimation, {
      toValue: 1,
      duration: 200,
      useNativeDriver: true,
    }).start();
  };

  return (
    <View className="items-center justify-center">
      <TouchableOpacity
        onPressIn={startRecording}
        onPressOut={stopRecording}
        className={`w-24 h-24 rounded-full flex items-center justify-center ${
          isRecording 
            ? 'bg-red-500 shadow-lg' 
            : isProcessing
            ? 'bg-yellow-500'
            : 'bg-purple-500 shadow-md'
        }`}
        disabled={isProcessing}
      >
        <Animated.View style={{ transform: [{ scale: pulseAnimation }] }}>
          {isRecording ? (
            <MicOff size={32} color="white" />
          ) : (
            <Mic size={32} color="white" />
          )}
        </Animated.View>
      </TouchableOpacity>
      
      <Text className="mt-4 text-gray-600 text-center">
        {isRecording 
          ? 'Listening to your question...' 
          : isProcessing
          ? 'Dewey is thinking...'
          : 'Hold to ask Dewey a question'
        }
      </Text>
    </View>
  );
};
```

### 2.2 Audio Playback Component

```typescript
// components/DeweyPlayer.tsx
import React, { useState, useEffect, useRef } from 'react';
import { View, TouchableOpacity, Text } from 'react-native';
import { Play, Pause, RotateCcw, Heart } from 'lucide-react-native';
import Sound from 'react-native-sound';

interface DeweyPlayerProps {
  audioData?: Uint8Array;
  question: string;
  onFavorite: () => void;
  onReplay: () => void;
}

export const DeweyPlayer: React.FC<DeweyPlayerProps> = ({
  audioData,
  question,
  onFavorite,
  onReplay
}) => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);
  const soundRef = useRef<Sound | null>(null);

  useEffect(() => {
    if (audioData) {
      setupAudio();
    }

    return () => {
      if (soundRef.current) {
        soundRef.current.release();
      }
    };
  }, [audioData]);

  const setupAudio = async () => {
    if (!audioData) return;

    try {
      // Convert PCM data to playable format
      const audioBlob = await convertPCMToWAV(audioData);
      const audioUrl = URL.createObjectURL(audioBlob);

      soundRef.current = new Sound(audioUrl, '', (error) => {
        if (error) {
          console.error('Audio setup error:', error);
          return;
        }
        
        setDuration(soundRef.current?.getDuration() || 0);
        // Auto-play Dewey's response
        playAudio();
      });
    } catch (error) {
      console.error('Error setting up audio:', error);
    }
  };

  const convertPCMToWAV = (pcmData: Uint8Array): Promise<Blob> => {
    return new Promise((resolve) => {
      const sampleRate = 24000; // Gemini outputs at 24kHz
      const numChannels = 1;
      const bytesPerSample = 2;
      
      const buffer = new ArrayBuffer(44 + pcmData.length);
      const view = new DataView(buffer);
      
      // WAV header
      const writeString = (offset: number, string: string) => {
        for (let i = 0; i < string.length; i++) {
          view.setUint8(offset + i, string.charCodeAt(i));
        }
      };
      
      writeString(0, 'RIFF');
      view.setUint32(4, 36 + pcmData.length, true);
      writeString(8, 'WAVE');
      writeString(12, 'fmt ');
      view.setUint32(16, 16, true);
      view.setUint16(20, 1, true);
      view.setUint16(22, numChannels, true);
      view.setUint32(24, sampleRate, true);
      view.setUint32(28, sampleRate * numChannels * bytesPerSample, true);
      view.setUint16(32, numChannels * bytesPerSample, true);
      view.setUint16(34, 8 * bytesPerSample, true);
      writeString(36, 'data');
      view.setUint32(40, pcmData.length, true);
      
      // Copy PCM data
      const uint8View = new Uint8Array(buffer, 44);
      uint8View.set(pcmData);
      
      resolve(new Blob([buffer], { type: 'audio/wav' }));
    });
  };

  const playAudio = () => {
    if (!soundRef.current) return;

    soundRef.current.play((success) => {
      if (success) {
        console.log('Audio playback completed');
      } else {
        console.error('Audio playback failed');
      }
      setIsPlaying(false);
      setProgress(0);
    });
    
    setIsPlaying(true);
    startProgressTracking();
  };

  const pauseAudio = () => {
    if (soundRef.current) {
      soundRef.current.pause();
      setIsPlaying(false);
    }
  };

  const startProgressTracking = () => {
    const interval = setInterval(() => {
      if (soundRef.current && isPlaying) {
        soundRef.current.getCurrentTime((seconds) => {
          const progressPercent = (seconds / duration) * 100;
          setProgress(progressPercent);
          
          if (seconds >= duration) {
            clearInterval(interval);
            setProgress(0);
            setIsPlaying(false);
          }
        });
      } else {
        clearInterval(interval);
      }
    }, 100);
  };

  return (
    <View className="bg-white rounded-xl p-4 shadow-sm">
      <Text className="text-gray-800 font-medium mb-3">
        Your Question: "{question}"
      </Text>
      
      <Text className="text-purple-700 font-medium mb-4">
        🎓 Dewey's Response:
      </Text>
      
      {/* Progress Bar */}
      <View className="bg-purple-200 rounded-full h-2 mb-4">
        <View 
          className="bg-purple-500 h-2 rounded-full"
          style={{ width: `${progress}%` }}
        />
      </View>
      
      {/* Controls */}
      <View className="flex-row items-center justify-between">
        <View className="flex-row items-center space-x-3">
          <TouchableOpacity
            onPress={isPlaying ? pauseAudio : playAudio}
            className="p-2 bg-purple-500 text-white rounded-full"
            disabled={!audioData}
          >
            {isPlaying ? (
              <Pause size={20} color="white" />
            ) : (
              <Play size={20} color="white" />
            )}
          </TouchableOpacity>
          
          <TouchableOpacity
            onPress={onReplay}
            className="p-2 bg-gray-100 rounded-full"
          >
            <RotateCcw size={20} color="#666" />
          </TouchableOpacity>
        </View>
        
        <TouchableOpacity
          onPress={onFavorite}
          className="p-2 bg-pink-100 rounded-full"
        >
          <Heart size={20} color="#EC4899" />
        </TouchableOpacity>
      </View>
    </View>
  );
};
```

## 3. Supabase Edge Function for Gemini Integration

```typescript
// supabase/functions/dewey-chat/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { corsHeaders } from '../_shared/cors.ts'

// Note: You'll need to install the Google GenAI SDK in your Supabase project
// This is a conceptual implementation - actual Edge Function integration may vary

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    const { audioData, userId, sessionId } = await req.json()
    
    // Initialize Gemini client
    const GEMINI_API_KEY = Deno.env.get('GEMINI_API_KEY')
    
    if (!GEMINI_API_KEY) {
      throw new Error('GEMINI_API_KEY not found')
    }

    // Create WebSocket connection to Gemini Live API
    const response = await establishGeminiSession(audioData, GEMINI_API_KEY)
    
    // Store the interaction in Supabase
    await storeInteraction(userId, sessionId, audioData, response.audioData)
    
    return new Response(
      JSON.stringify({ 
        audioResponse: response.audioData,
        transcription: response.transcription,
        sessionId: sessionId 
      }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
        status: 200,
      },
    )

  } catch (error) {
    console.error('Dewey chat error:', error)
    
    return new Response(
      JSON.stringify({ error: error.message }),
      {
        headers: { ...corsHeaders, 'Content-Type': 'application/json' },
        status: 500,
      },
    )
  }
})

async function establishGeminiSession(audioData: Uint8Array, apiKey: string) {
  // This is a conceptual implementation
  // The actual WebSocket connection in Deno may require different approach
  
  const config = {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      contents: [{
        parts: [{
          audio: {
            rawPcm: Array.from(audioData),
            sampleRate: 16000
          }
        }]
      }],
      systemInstruction: {
        parts: [{
          text: `You are Dewey, a wise and curious AI companion named after educator John Dewey. 
          
          Your mission is to nurture children's natural curiosity through engaging, podcast-style conversations.
          
          Guidelines:
          - Speak warmly and enthusiastically, like a favorite teacher
          - Break complex ideas into simple, age-appropriate explanations (ages 6-12)
          - Use imaginative examples and metaphors children can relate to
          - Connect learning to real-world experiences kids know
          - Always validate their curiosity before answering
          - End with questions that encourage deeper exploration
          - Keep responses to 2-4 engaging paragraphs
          - Use a natural, conversational speaking style perfect for audio
          
          Remember: You're not just answering questions - you're modeling curiosity and critical thinking!`
        }]
      },
      generationConfig: {
        maxOutputTokens: 2048,
        temperature: 0.7
      }
    })
  };

  // Note: Real implementation would use WebSocket for streaming
  const response = await fetch(
    'https://generativelanguage.googleapis.com/v1/models/gemini-live-2.5-flash-preview-native-audio:generateContent',
    config
  );

  const result = await response.json();
  
  return {
    audioData: result.candidates[0].content.parts[0].audio?.rawPcm || null,
    transcription: result.candidates[0].content.parts[0].text || ''
  };
}

async function storeInteraction(userId: string, sessionId: string, questionAudio: Uint8Array, responseAudio: Uint8Array) {
  // Store interaction in Supabase database
  const { data, error } = await supabaseClient
    .from('questions')
    .insert({
      user_id: userId,
      session_id: sessionId,
      question_audio_data: questionAudio,
      answer_audio_data: responseAudio,
      created_at: new Date().toISOString()
    });

  if (error) {
    console.error('Database error:', error);
    throw error;
  }

  return data;
}
```

## 4. Main App Integration

```typescript
// screens/AskDeweyScreen.tsx
import React, { useState } from 'react';
import { View, Text, ScrollView } from 'react-native';
import { DeweyMicrophone } from '../components/DeweyMicrophone';
import { DeweyPlayer } from '../components/DeweyPlayer';
import { useSupabase } from '../contexts/SupabaseContext';

export const AskDeweyScreen: React.FC = () => {
  const [currentQuestion, setCurrentQuestion] = useState<string>('');
  const [audioResponse, setAudioResponse] = useState<Uint8Array | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const { supabase, user } = useSupabase();

  const handleAudioData = async (audioData: Uint8Array) => {
    setIsLoading(true);
    
    try {
      // Call Supabase Edge Function
      const { data, error } = await supabase.functions.invoke('dewey-chat', {
        body: {
          audioData: Array.from(audioData),
          userId: user?.id,
          sessionId: generateSessionId()
        }
      });

      if (error) throw error;

      setAudioResponse(new Uint8Array(data.audioResponse));
      setCurrentQuestion(data.transcription || "Your question");
      
    } catch (error) {
      console.error('Error calling Dewey:', error);
    } finally {
      setIsLoading(false);
    }
  };

  const handleFavorite = async () => {
    if (!currentQuestion || !user) return;

    try {
      await supabase
        .from('favorites')
        .insert({
          user_id: user.id,
          question_text: currentQuestion,
          audio_data: audioResponse
        });
        
      // Show success message
      console.log('Added to favorites!');
    } catch (error) {
      console.error('Error adding to favorites:', error);
    }
  };

  const generateSessionId = () => {
    return `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  };

  return (
    <ScrollView className="flex-1 bg-gradient-to-b from-purple-50 to-blue-50">
      <View className="p-6">
        {/* Header */}
        <View className="text-center mb-8">
          <Text className="text-3xl font-bold text-gray-800 mb-2">
            🎓 Ask Dewey!
          </Text>
          <Text className="text-gray-600">
            Your curious learning companion
          </Text>
        </View>

        {/* Microphone Interface */}
        {!audioResponse && (
          <View className="items-center mb-8">
            <DeweyMicrophone
              onAudioData={handleAudioData}
              onRecordingStart={() => console.log('Recording started')}
              onRecordingEnd={() => console.log('Recording ended')}
            />
            
            {isLoading && (
              <View className="mt-4 p-4 bg-yellow-100 rounded-lg">
                <Text className="text-yellow-800 text-center">
                  🤔 Dewey is thinking about your question...
                </Text>
              </View>
            )}
          </View>
        )}

        {/* Audio Response Player */}
        {audioResponse && currentQuestion && (
          <DeweyPlayer
            audioData={audioResponse}
            question={currentQuestion}
            onFavorite={handleFavorite}
            onReplay={() => {
              setAudioResponse(null);
              setCurrentQuestion('');
            }}
          />
        )}

        {/* Educational Tips */}
        <View className="mt-8 p-4 bg-white rounded-xl shadow-sm">
          <Text className="font-semibold text-gray-800 mb-2">
            💡 Great Questions to Ask Dewey:
          </Text>
          <Text className="text-gray-600 text-sm leading-relaxed">
            • "Why is the sky blue?" {'\n'}
            • "How do airplanes stay up in the air?" {'\n'}
            • "Why do we need to sleep?" {'\n'}
            • "How do plants grow?" {'\n'}
            • "Why does ice melt?"
          </Text>
        </View>
      </View>
    </ScrollView>
  );
};
```

## 5. Key Implementation Notes

### Audio Handling Considerations:
- **Format Conversion:** Gemini expects 16kHz PCM input, outputs 24kHz PCM
- **Real-time Streaming:** Use WebSocket for bidirectional audio streaming
- **Latency Optimization:** Buffer audio chunks for smooth playback
- **Error Recovery:** Handle network interruptions gracefully

### Security Best Practices:
- **API Key Protection:** Store Gemini API key in Supabase environment variables
- **User Authentication:** Validate user sessions before API calls
- **Rate Limiting:** Implement usage limits to prevent abuse
- **Content Filtering:** Monitor responses for inappropriate content

### Performance Optimizations:
- **Audio Caching:** Cache frequently accessed audio responses
- **Compression:** Compress audio data for storage efficiency
- **Background Processing:** Handle audio processing in background threads
- **Memory Management:** Clean up audio resources after playback

### Development Workflow:
1. Set up Gemini API access and test basic audio streaming
2. Implement React Native audio recording/playback
3. Create Supabase Edge Function for API integration
4. Build UI components with proper error handling
5. Test end-to-end audio pipeline
6. Implement caching and optimization features

This technical implementation provides a solid foundation for building Dewey's voice-first educational experience using the Gemini 2.5 Flash Live API!