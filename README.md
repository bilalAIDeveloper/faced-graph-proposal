# StepMate Facet-Graph Onboarding Architecture
## Technical Implementation Guide

---

## 🏗️ Architecture Overview

**Goal**: Replace fixed 5-stage onboarding with dynamic, mission-driven facet collection  
**Approach**: Configuration-driven architecture using existing chat infrastructure  
**Key Change**: Dynamic prompt generation based on user's current phase and mission

---

## 📁 File Structure

```
backend/CircleServer/functions/src/
├── config/
│   ├── facets.config.js          # Facet definitions & validation
│   ├── missions.config.js         # Mission types & requirements  
│   ├── onboarding.config.js       # Flow configuration
│   └── state.config.js           # Enhanced existing config
├── services/chat/
│   └── chat.service.js           # Enhanced with onboarding detection
└── prompts/chat/
    └── unified.prompt.js         # Dynamic prompt generation
```

---

## 🔧 Configuration Files

### 1. `facets.config.js` - Facet Registry

```javascript
const FACET_VALIDATORS = {
  location: {
    type: 'location',
    required: true,
    standardization: (value) => value.includes('USA') ? value : `${value}, USA`
  },
  gender: {
    type: 'enum',
    values: ['female', 'male', 'non-binary', 'prefer_not_to_say'],
    required: true
  },
  step: {
    type: 'enum',
    values: ['1', '2', '3'],
    standardization: (value) => value.replace('Step ', '')
  },
  budget: {
    type: 'band',
    values: ['$', '$$', '$$$', '$$$$'],
    standardization: (value) => {
      const amount = parseInt(value.replace(/[^0-9]/g, ''));
      if (amount <= 30) return '$';
      if (amount <= 60) return '$$';
      if (amount <= 100) return '$$$';
      return '$$$$';
    }
  }
};

const FACET_PACKS = {
  core: ['location', 'gender', 'commsPref', 'timezoneBand'],
  student: ['step', 'studyStyle', 'studyHours', 'intent'],
  tutor: ['stepsTaught', 'specialties', 'experienceYrs', 'ratesBand'],
  resident: ['specialty', 'pgy', 'mentoring', 'availabilityBands']
};
```

### 2. `missions.config.js` - Mission Definitions

```javascript
const MISSIONS = {
  student_peer: {
    id: 'student_peer',
    name: 'Find Study Partner',
    roleCapabilities: ['student'],
    requiredFacets: {
      core: ['location', 'gender', 'commsPref'],
      student: ['step', 'studyStyle', 'intent']
    }
  },
  tutor: {
    id: 'tutor',
    name: 'Find Tutor',
    roleCapabilities: ['student'],
    requiredFacets: {
      core: ['location', 'gender', 'commsPref'],
      student: ['step', 'budget'],
      tutor: ['stepsTaught', 'specialties']
    }
  }
};
```

### 3. `onboarding.config.js` - Flow Configuration

```javascript
const ONBOARDING_PHASES = {
  core_facets: {
    id: 'core_facets',
    requiredFacets: ['location', 'gender', 'commsPref'],
    completionThreshold: 0.75,
    nextPhase: 'mission_selection'
  },
  mission_selection: {
    id: 'mission_selection',
    requiredFacets: ['selectedMission'],
    completionThreshold: 1.0,
    nextPhase: 'role_facets'
  },
  role_facets: {
    id: 'role_facets',
    requiredFacets: [], // Dynamic based on mission
    completionThreshold: 0.7,
    nextPhase: 'mission_overrides'
  }
};

const CORE_FACETS_COLLECTION = {
  collectionOrder: ['location', 'gender', 'commsPref', 'timezoneBand'],
  questions: {
    location: { text: "Where are you located?" },
    gender: { text: "What's your gender?", options: ['female', 'male', 'non-binary'] },
    commsPref: { text: "How do you prefer to communicate?", options: ['text', 'video', 'audio'] }
  }
};
```

### 4. Enhanced `state.config.js` - Integration Layer

```javascript
// Import new facet system configurations
const { FACET_VALIDATORS } = require('./facets.config');
const { MISSIONS } = require('./missions.config');
const { CORE_FACETS_COLLECTION } = require('./onboarding.config');

// Enhanced STAGES - Replace old 5-stage onboarding with 4 dynamic phases
const STAGES = {
  welcome: { /* existing welcome stage */ },
  
  // NEW: 4 distinct onboarding phases
  core_facets: {
    id: 'core_facets',
    completionCriteria: { type: 'facet_based', requiredFacets: ['location', 'gender', 'commsPref'] }
  },
  mission_selection: {
    id: 'mission_selection', 
    completionCriteria: { type: 'mission_based', requiredAnswers: ['selectedMission'] }
  },
  role_facets: {
    id: 'role_facets',
    completionCriteria: { type: 'facet_based', requiredFacets: [] } // Dynamic based on mission
  },
  mission_overrides: {
    id: 'mission_overrides',
    completionCriteria: { type: 'optional_based' } // Optional phase
  }
};

// NEW: Dynamic question generation
const DYNAMIC_QUESTIONS = {
  generateForPhase: (phase, userData) => {
    switch (phase) {
      case 'core_facets': return generateCoreQuestions(userData);
      case 'mission_selection': return generateMissionQuestions(userData);
      case 'role_facets': return generateRoleQuestions(userData);
      default: return [];
    }
  }
};

module.exports = { STAGES, DYNAMIC_QUESTIONS, COMPLETION_LOGIC };
```

---

## 🔄 Enhanced Chat Service

### Updated `chat.service.js`

```javascript
class ChatService {
  async handleChatMessage(userId, message, conversationHistory) {
    const userStatus = await this.getUserStatus(userId);
    
    if (userStatus.isOnboarding) {
      return await this.handleOnboardingChat(userId, message, conversationHistory, userStatus);
    } else {
      return await this.handlePostOnboardingChat(userId, message, conversationHistory, userStatus);
    }
  }

  // NEW: Onboarding Chat Handler
  async handleOnboardingChat(userId, message, conversationHistory, userStatus) {
    const { currentPhase, selectedMission, completedFacets } = userStatus;
    
    const prompt = generateUnifiedPrompt({
      messages: conversationHistory,
      isOnboarding: true,
      currentPhase: currentPhase,
      selectedMission: selectedMission,
      userData: { facets: completedFacets, selectedMission }
    });

    const aiResponse = await this.processWithAI(prompt, message);
    return await this.handleOnboardingResponse(userId, aiResponse, userStatus);
  }

  // NEW: Enhanced user status detection
  async getUserStatus(userId) {
    const userDoc = await db.collection('users').doc(userId).get();
    
    if (!userDoc.exists) {
      return {
        isOnboarding: true,
        currentPhase: 'core_facets',
        selectedMission: null,
        completedFacets: {},
        roleCapabilities: []
      };
    }

    const userData = userDoc.data();
    const facets = userData.facets || {};
    const selectedMission = userData.selectedMission;
    
    const isOnboardingComplete = this.isOnboardingComplete(facets, selectedMission);
    
    if (isOnboardingComplete) {
      return {
        isOnboarding: false,
        facets: facets,
        selectedMission: selectedMission,
        roleCapabilities: this.getRoleCapabilities(selectedMission)
      };
    } else {
      return {
        isOnboarding: true,
        currentPhase: this.determineCurrentPhase(facets, selectedMission),
        selectedMission: selectedMission,
        completedFacets: facets
      };
    }
  }
}
```

---

## 🎯 Dynamic Prompt Generation

### Enhanced `unified.prompt.js`

```javascript
const generateUnifiedPrompt = ({ messages, isOnboarding = false, currentPhase = null, selectedMission = null, userData = {} }) => {
  const onboardingContext = isOnboarding ? getOnboardingContext(currentPhase, selectedMission) : null;
  
  // NEW: Get dynamic questions for current phase
  const currentQuestions = isOnboarding ? getDynamicQuestionsForPrompt(currentPhase, userData) : [];
  const questionContext = isOnboarding ? embedQuestionsIntoPrompt(currentQuestions, userData) : '';

  const dynamicContext = `
## Current State
- Is Onboarding: ${isOnboarding}
${onboardingContext ? `- Current Phase: ${onboardingContext.phase.name}
- Required Facets: ${onboardingContext.requiredFacets.join(', ')}` : ''}
- Current User Profile:
${JSON.stringify(messages, null, 2)}

${questionContext}
`;

  const instructions = `
You are StepMate, a USMLE study buddy matcher. Your job is to:
${isOnboarding ? getOnboardingInstructions(currentPhase, selectedMission) : getRegularChatInstructions()}

✅ Call the appropriate tool based on current context:
${isOnboarding ? `
- updateFacetState: When extracting facet values during onboarding
- selectMission: When user indicates their mission preference
- transitionPhase: When current phase is complete
- completeOnboarding: When all required facets are collected` : `
- updateState: When extracting onboarding state updates (existing functionality)
- findMatches: When user requests matching
- updatePreferences: When user updates preferences`}
`;

  return { dynamicContext, instructions };
};
```

---

## 🛠️ AI Tools & Functions

### Onboarding Tools

```javascript
// Tool: Update Facet State
const updateFacetState = {
  name: 'updateFacetState',
  description: 'Update user facet values during onboarding',
  parameters: {
    type: 'object',
    properties: {
      facet: { type: 'string', description: 'The facet being updated (e.g., location, gender, step)' },
      value: { type: 'string', description: 'The value provided by the user' },
      facetType: { type: 'string', enum: ['core', 'role'], description: 'Core or role-specific facet' },
      role: { type: 'string', enum: ['student', 'tutor', 'resident'], description: 'Role for role-specific facets' }
    },
    required: ['facet', 'value', 'facetType']
  }
};

// Tool: Select Mission
const selectMission = {
  name: 'selectMission',
  description: 'Select user\'s primary mission/intent',
  parameters: {
    type: 'object',
    properties: {
      mission: { type: 'string', enum: ['student_peer', 'tutor', 'student', 'resident'] },
      confidence: { type: 'number', minimum: 0, maximum: 1, description: 'Confidence level (0-1)' }
    },
    required: ['mission', 'confidence']
  }
};

// Tool: Transition Phase
const transitionPhase = {
  name: 'transitionPhase',
  description: 'Transition to next onboarding phase when current phase is complete',
  parameters: {
    type: 'object',
    properties: {
      fromPhase: { type: 'string', enum: ['core_facets', 'mission_selection', 'role_facets', 'mission_overrides'] },
      toPhase: { type: 'string', enum: ['mission_selection', 'role_facets', 'mission_overrides', 'complete'] },
      reason: { type: 'string', description: 'Reason for phase transition' }
    },
    required: ['fromPhase', 'toPhase', 'reason']
  }
};

// Tool: Complete Onboarding
const completeOnboarding = {
  name: 'completeOnboarding',
  description: 'Mark onboarding as complete when all required facets are collected',
  parameters: {
    type: 'object',
    properties: {
      selectedMission: { type: 'string', description: 'The final selected mission' },
      roleCapabilities: { type: 'array', items: { type: 'string' }, description: 'Inferred role capabilities' },
      summary: { type: 'string', description: 'Summary of collected user profile' }
    },
    required: ['selectedMission', 'roleCapabilities', 'summary']
  }
};
```

### Tool Usage Examples

```javascript
// Example 1: User provides location during core facets
updateFacetState({
  facet: 'location',
  value: 'Seattle, WA',
  facetType: 'core'
});

// Example 2: User indicates they want a tutor
selectMission({
  mission: 'tutor',
  confidence: 0.9
});

// Example 3: Core facets complete, transition to mission selection
transitionPhase({
  fromPhase: 'core_facets',
  toPhase: 'mission_selection',
  reason: 'All core facets collected'
});

// Example 4: Onboarding complete
completeOnboarding({
  selectedMission: 'tutor',
  roleCapabilities: ['student'],
  summary: 'User in Seattle seeking Step 1 tutor with $50-100 budget'
});
```

---

## 🔄 User Experience Flow

### Phase 1: Core Facets Collection
```
AI: "Tell me about yourself - where are you located?"
User: "I'm in Seattle, Washington"
AI: "What's your gender?"
User: "Female"
AI: "How do you prefer to communicate?"
User: "Video calls and text messages"
```

### Phase 2: Mission Selection (Contextualized)
```
AI: "Great! I see you're in Seattle and prefer video calls. 
     Who are you looking for right now?"
User: "I need a tutor for Step 1"
AI: "Perfect! I'll help you find Step 1 tutors in Seattle 
     who offer video sessions."
```

### Phase 3: Role-Specific Facets
```
AI: "Which USMLE Step are you preparing for?"
User: "Step 1"
AI: "What's your budget range for Step 1 tutoring in Seattle?"
User: "Around $50-100 per hour"
```


## 📊 Key Benefits

### Technical Benefits
- **Configuration-Driven**: Easy to modify facets, missions, or flows without code changes
- **Dynamic Prompts**: AI questions adapt based on current phase and mission
- **Scalable Architecture**: Easy to add new roles, facets, or missions
- **Backward Compatible**: Existing chat functionality preserved

### User Experience Benefits
- **Contextual Questions**: AI references user's profile for personalized questions
- **Progressive Flow**: Only asks relevant questions in optimal order
- **Natural Conversations**: Questions feel more conversational and contextual
- **Efficient Onboarding**: Reduces time and improves completion rates

---

*This architecture transforms StepMate's onboarding from a rigid 5-stage process to a dynamic, mission-driven experience while leveraging existing chat infrastructure.*
