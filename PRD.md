# Dewey - Voice AI App for Curious Kids
## Product Requirements Document (PRD)

---

## 1. Executive Summary

**Product Name:** Dewey  
**Tagline:** "Ask Dewey Anything!"  
**Platform:** React Native (iOS & Android)  
**Target Audience:** Children ages 6-12 and their parents  
**Core Value:** A voice-first AI companion named after educational pioneer John Dewey that satisfies children's natural curiosity through engaging, educational podcast-style conversations powered by Google's Gemini 2.5 Flash Live API.

---

## 2. Product Vision & Objectives

### Vision Statement
To create the world's most engaging voice-first educational platform that nurtures children's curiosity and makes learning as natural as having a conversation with a knowledgeable friend.

### Primary Objectives
- **Educational Impact:** Provide accurate, age-appropriate explanations to children's "why" and "how" questions
- **Engagement:** Maintain high user engagement through conversational, podcast-style interactions
- **Safety:** Ensure all content is child-safe with robust moderation and parental controls
- **Accessibility:** Make learning accessible through voice-first interface that doesn't require reading skills
- **Growth:** Build a platform that scales with user-generated content and community engagement

---

## 3. Target Audience

### Primary Users
**Children (Ages 6-12)**
- Naturally curious with endless questions
- Prefer interactive, audio-based content
- Limited reading skills but strong listening comprehension
- Enjoy story-telling and imaginative explanations

### Secondary Users
**Parents/Guardians**
- Seeking educational screen-time alternatives
- Want to encourage their child's curiosity
- Need tools to manage and monitor content
- Value educational apps that promote learning

### User Personas

#### Persona 1: "Curious Emma" (Age 8)
- Loves asking "why" about everything she sees
- Enjoys podcasts and audio stories
- Gets frustrated with text-heavy apps
- Learns best through stories and examples

#### Persona 2: "Supportive Parent Sarah" (Age 35)
- Working mother seeking quality educational content
- Wants to encourage Emma's questions but doesn't always have time/answers
- Concerned about screen time but values educational technology
- Needs easy content monitoring and parental controls

---

## 4. Core Features & User Stories

### 4.1 Voice-First Question & Answer (MVP)
**User Story:** "As a curious child, I want to ask questions using my voice and hear engaging explanations so that I can learn about the world around me."

**Features:**
- Real-time voice recording with visual feedback
- Natural language processing via Gemini 2.5 Flash Live API
- Podcast-style audio responses (2-4 paragraphs)
- Child-friendly explanations with metaphors and examples
- Background sound effects and engaging delivery

**Technical Requirements:**
- React Native Voice module for audio recording
- Integration with Gemini Live API for bidirectional audio
- Audio playback controls (play, pause, replay)
- Offline audio caching for favorites

### 4.2 Favorites System (MVP)
**User Story:** "As a child, I want to save my favorite questions and answers so I can listen to them again and share them with friends and family."

**Features:**
- One-tap "save to favorites" during or after playback
- Favorites tab with organized list view
- Quick replay functionality
- Share favorites via standard sharing APIs

**Technical Requirements:**
- Supabase storage for user favorites
- Audio file management and caching
- User authentication for data persistence

### 4.3 Explore Feed (MVP)
**User Story:** "As a child, I want to discover what other kids are asking about so I can learn new things I never thought to ask."

**Features:**
- Curated feed of popular questions from other children
- Age-appropriate content filtering
- Questions attributed to "Emma, age 7" format
- Ability to "favorite" questions from explore feed

**Technical Requirements:**
- Supabase database for community questions
- Content moderation system
- Trending/popular question algorithms
- Real-time updates for fresh content

### 4.4 Parental Dashboard (Post-MVP)
**User Story:** "As a parent, I want to see what my child is learning about and ensure the content is appropriate."

**Features:**
- Child's question history and listening time
- Content appropriateness controls
- Usage time limits and scheduling
- Educational progress insights
- Report inappropriate content

**Technical Requirements:**
- Parent account creation and authentication
- Child profile management
- Usage analytics and reporting
- Content filtering preferences

---

## 5. Technical Architecture

### 5.1 Frontend - React Native

**Core Libraries:**
```json
{
  "@react-native-voice/voice": "^3.2.4",
  "@react-native-async-storage/async-storage": "^1.19.0",
  "react-native-track-player": "^4.0.0",
  "@react-navigation/native": "^6.1.0",
  "@react-navigation/bottom-tabs": "^6.5.0",
  "react-native-paper": "^5.10.0",
  "@supabase/supabase-js": "^2.38.0",
  "react-native-share": "^10.0.0",
  "react-native-permissions": "^3.9.0",
  "@google/genai": "^latest",
  "react-native-sound": "^0.11.2"
}
```

**Real-Time Audio Architecture:**
- **WebSocket Connection:** Direct integration with Gemini 2.5 Flash Live API
- **Audio Format:** 16kHz PCM input, 24kHz PCM output 
- **Native Audio Processing:** Real-time bidirectional streaming
- **Model:** `gemini-live-2.5-flash-preview-native-audio` for enhanced voice quality

### 5.2 Backend - Supabase + Edge Functions

**Database Schema:**
```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE,
  child_name TEXT,
  child_age INTEGER,
  parent_email TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Questions table
CREATE TABLE questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  question_text TEXT NOT NULL,
  question_audio_url TEXT,
  answer_text TEXT,
  answer_audio_url TEXT,
  duration INTEGER, -- in seconds
  is_public BOOLEAN DEFAULT false,
  is_moderated BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT NOW(),
  child_age INTEGER,
  child_first_name TEXT
);

-- Favorites table
CREATE TABLE favorites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  question_id UUID REFERENCES questions(id),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, question_id)
);

-- Usage analytics
CREATE TABLE usage_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  session_start TIMESTAMP DEFAULT NOW(),
  session_end TIMESTAMP,
  questions_asked INTEGER DEFAULT 0,
  total_listening_time INTEGER DEFAULT 0
);
```

**Edge Functions:**
```typescript
// /functions/dewey-chat/index.ts
import { GoogleGenAI, Modality } from '@google/genai';

export async function deweyChat(request: Request) {
  const { audioData, userId, sessionId } = await request.json();
  
  const ai = new GoogleGenAI({});
  const modelName = "gemini-live-2.5-flash-preview-native-audio";
  
  const config = {
    responseModalities: [Modality.AUDIO],
    systemInstruction: DEWEY_PROMPT
  };

  const session = await ai.live.connect({
    model: modelName,
    config: config,
    callbacks: {
      onopen: () => console.log('Dewey session opened'),
      onclose: () => console.log('Dewey session closed'),
      onerror: (error) => console.error('Dewey error:', error)
    }
  });

  // Send audio input
  await session.sendClientContent({
    turns: { 
      role: "user", 
      parts: [{ audio: { rawPcm: audioData } }] 
    },
    turnComplete: true
  });

  // Collect audio response
  const audioChunks = [];
  for await (const response of session.receive()) {
    if (response.data !== undefined) {
      audioChunks.push(response.data);
    }
    if (response.done) break;
  }

  // Store interaction in database
  await storeInteraction(userId, audioData, audioChunks);
  
  session.close();
  return { audioResponse: combineAudioChunks(audioChunks) };
}

const DEWEY_PROMPT = `
You are Dewey, a wise and curious AI companion named after the great educator John Dewey. Your mission is to nurture children's natural curiosity through engaging, podcast-style conversations.

Core Principles:
- Break down complex ideas into simple, friendly explanations for ages 6-12
- Use imaginative examples and metaphors kids relate to
- Encourage wonder and make learning fun with a warm, positive tone
- Stay factually accurate but never speak down to your audience
- Connect to everyday experiences children know
- Use light humor and playful language to create a lively audio atmosphere
- Always invite more questions at the end

Response Format:
- Speak as if hosting a podcast for curious kids
- Keep responses to 2-4 engaging paragraphs
- Use clear, simple sentences with natural speech patterns
- Include sound effects like [birds chirping] when appropriate

Example Response Style:
"Wow, what a fantastic question! You know, when I think about why the sky is blue, I imagine sunlight is like a big box of crayons with all different colors mixed together. When this colorful sunlight hits the tiny particles floating in our air, something magical happens - the blue color bounces around everywhere, like a bouncy ball in a playground! That's why we see blue when we look up at the sky.

Isn't that amazing? The next time you see a beautiful blue sky, you'll know it's because sunlight is playing catch with those tiny particles up there! What else makes you curious about our amazing world?"
`;
```

### 5.3 Do We Need Node/Express Backend?

**Recommendation: No separate Node/Express backend needed.**

**Rationale:**
1. **Supabase Edge Functions** handle API integrations and business logic
2. **Direct Gemini API calls** can be made from Edge Functions
3. **Real-time features** provided by Supabase subscriptions
4. **Authentication** handled by Supabase Auth
5. **File storage** managed by Supabase Storage
6. **Reduced complexity** and deployment overhead

**Edge Function Benefits:**
- Serverless scaling
- Global edge deployment
- TypeScript support
- Integrated with Supabase ecosystem
- Cost-effective for moderate usage

---

## 6. User Experience (UX) Flow

### 6.1 Onboarding Flow
1. **Welcome Screen** - App intro and value proposition
2. **Child Setup** - Name, age, and voice permissions
3. **Parent Setup** - Optional parent email for progress updates
4. **Permission Requests** - Microphone and storage permissions
5. **First Question** - Guided experience asking first question

### 6.2 Core User Journey - Asking a Question
1. **Home Screen** - Large microphone button with "Ask a Question"
2. **Recording State** - Visual feedback (pulsing animation, sound waves)
3. **Processing State** - "Thinking..." with engaging animations
4. **Audio Response** - Playback controls and "Save to Favorites" option
5. **Related Suggestions** - "You might also like..." recommendations

### 6.3 Navigation Structure
```
Bottom Tab Navigation:
├── Ask (Home)
│   ├── Microphone Interface
│   ├── Current Question Player
│   └── Recent Questions
├── Favorites
│   ├── Saved Questions List
│   ├── Search Favorites
│   └── Share Options
└── Explore
    ├── Trending Questions
    ├── Categories (Animals, Science, etc.)
    └── Age-based Filters
```

---

## 7. Content Strategy & Safety

### 7.1 Content Guidelines
- **Age-Appropriate Language:** Simple vocabulary suitable for ages 6-12
- **Educational Focus:** Scientifically accurate with imaginative explanations
- **Positive Tone:** Encouraging curiosity and wonder
- **Cultural Sensitivity:** Inclusive examples and perspectives
- **Length Control:** 2-4 paragraph responses (2-3 minutes audio)

### 7.2 Safety Measures
- **Content Moderation:** Automated filtering + human review for public content
- **Question Filtering:** Block inappropriate topics automatically
- **Privacy Protection:** No sharing of personal information
- **Parental Controls:** Content reporting and time management
- **Data Protection:** COPPA compliant data handling

### 7.3 Dewey's Educational Persona
```typescript
const DEWEY_SYSTEM_PROMPT = `
You are Dewey, a wise and curious AI companion named after the great educator John Dewey, who believed that children learn best through experience and inquiry. Your mission is to nurture children's natural curiosity through engaging, podcast-style conversations.

John Dewey's Educational Philosophy (Your Foundation):
- "Learn by doing" - connect explanations to hands-on experiences
- Follow the child's natural curiosity and interests
- Make learning a collaborative, social experience
- Connect knowledge to real-world applications

Your Personality as Dewey:
- Wise but playful, like a favorite teacher who makes everything interesting
- Endlessly curious yourself - you love learning alongside children
- Patient and encouraging, never making children feel silly for asking
- Enthusiastic about discovery and "aha!" moments

Communication Style:
- Speak as if hosting an engaging educational podcast
- Use the "Dewey Method" of learning through questioning
- Break complex ideas into simple, memorable concepts
- Always validate the child's curiosity before answering
- End with questions that encourage deeper exploration

Response Structure:
1. Celebrate the question ("What a wonderful thing to wonder about!")
2. Provide 2-4 paragraphs of age-appropriate explanation
3. Use relatable examples and metaphors
4. Invite further exploration ("What else makes you curious about...")

Remember: You're not just answering questions - you're modeling how to think, explore, and stay curious about the world. Every response should inspire the child to keep asking and learning.
`;
```

---

## 8. Monetization Strategy

### 8.1 Freemium Model
**Free Tier:**
- 10 questions per day
- Access to favorites
- Browse explore feed
- Basic parental controls

**Premium Tier ($4.99/month or $29.99/year):**
- Unlimited questions
- Advanced favorites organization (categories, notes)
- Exclusive premium content
- Enhanced parental dashboard
- Priority customer support
- Offline listening for favorites

### 8.2 Future Revenue Streams
- **Educational Partnerships:** Content collaboration with museums, science centers
- **White-label Licensing:** Customize app for schools and educational institutions
- **Merchandise:** Character-based learning materials and books
- **Live Events:** Virtual Q&A sessions with experts

---

## 9. Success Metrics & KPIs

### 9.1 Engagement Metrics
- **Daily Active Users (DAU):** Target 40% of registered users
- **Session Length:** Average 15-20 minutes per session
- **Questions per Session:** Average 3-5 questions
- **Retention:** 70% Week 1, 50% Month 1, 30% Month 3

### 9.2 Educational Impact
- **Question Diversity:** Number of unique topics explored
- **Learning Depth:** Return to related topics and follow-up questions
- **Parent Satisfaction:** Surveys and app store ratings
- **Content Quality:** Audio completion rates and favorites ratio

### 9.3 Business Metrics
- **User Acquisition Cost (CAC):** Target <$10 per user
- **Lifetime Value (LTV):** Target >$30 per user
- **Conversion Rate:** Free to premium conversion >8%
- **Churn Rate:** Monthly churn <5% for premium users

---

## 10. Development Timeline

### Phase 1: MVP Development (12 weeks)
**Weeks 1-4: Foundation**
- React Native app setup and basic navigation
- Supabase project configuration
- Voice recording and playback implementation
- Basic UI/UX implementation

**Weeks 5-8: Core Features**
- Gemini 2.5 Flash Live API integration
- Favorites system implementation
- Explore feed with mock data
- Audio caching and offline support

**Weeks 9-12: Polish & Testing**
- Content moderation system
- Performance optimization
- Beta testing with target families
- App store submission preparation

### Phase 2: Enhanced Features (8 weeks)
**Weeks 13-16:**
- Parental dashboard implementation
- Advanced content filtering
- Push notifications
- Analytics integration

**Weeks 17-20:**
- Premium subscription system
- Enhanced UI/UX polish
- Performance optimization
- Marketing website launch

### Phase 3: Scale & Growth (Ongoing)
- Community features
- Content partnerships
- Advanced AI features
- International expansion

---

## 11. Technical Considerations & Risks

### 11.1 Scalability Concerns
- **API Rate Limits:** Gemini API quotas for high usage
- **Audio Storage Costs:** Supabase storage scaling
- **Real-time Performance:** Voice processing latency
- **Mitigation:** Caching strategies, CDN usage, usage monitoring

### 11.2 Privacy & Compliance
- **COPPA Compliance:** Children's data protection requirements
- **GDPR Considerations:** European user data handling
- **Audio Data Security:** Encrypted storage and transmission
- **Mitigation:** Legal review, privacy-by-design architecture

### 11.3 Content Quality Control
- **AI Response Accuracy:** Potential for incorrect information
- **Inappropriate Content:** Risk of unsuitable responses
- **Cultural Bias:** AI model biases affecting responses
- **Mitigation:** Human oversight, community reporting, continuous monitoring

---

## 12. Go-to-Market Strategy

### 12.1 Launch Strategy
1. **Private Beta** - 50 families for feedback and iteration
2. **App Store Soft Launch** - Release in select English-speaking markets
3. **Influencer Partnerships** - Parent bloggers and educational content creators
4. **PR Campaign** - Tech and parenting media outreach

### 12.2 Marketing Channels
- **Organic:** App Store Optimization (ASO) and content marketing
- **Paid:** Meta and Google Ads targeting parents of young children
- **Partnerships:** Educational toy companies and children's content platforms
- **Community:** Parent forums and educational communities

### 12.3 Success Criteria for Launch
- 1,000 downloads in first month
- 4.5+ app store rating
- 60%+ user retention after one week
- 20+ positive reviews mentioning educational value

---

## Conclusion

Curious Kids represents a unique opportunity to combine cutting-edge AI technology with educational content to create a product that genuinely enhances children's learning experiences. The voice-first approach removes barriers to engagement while the community features create a scalable content ecosystem.

The technical architecture leveraging React Native and Supabase provides a robust, scalable foundation while keeping development complexity manageable. The integration with Gemini 2.5 Flash Live API offers cost-effective, high-quality AI responses that can grow with our user base.

Success will depend on execution quality, community growth, and continuous iteration based on user feedback. The monetization strategy provides multiple revenue streams while keeping the core experience accessible to all families.

**Next Steps:**
1. Validate technical feasibility with Gemini API testing
2. Create detailed wireframes and user flow documentation  
3. Begin development with core voice recording and playback features
4. Establish beta testing program with target families