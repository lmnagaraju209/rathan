# Video Script Template
## Use this template for each video in your Udemy course

---

## Video: [Module Number].[Video Number] - [Video Title]

**Duration:** [X minutes]  
**Module:** [Module Name]  
**Learning Objectives:**
- [ ] Objective 1
- [ ] Objective 2
- [ ] Objective 3

---

## Script Outline

### Introduction (30 seconds)
- [ ] Greet viewers
- [ ] Recap previous video (if applicable)
- [ ] State what you'll learn in this video
- [ ] Show end result (what they'll have after video)

**Script:**
"Hey everyone, welcome back! In the last video, we [brief recap]. Today, we're going to [what you'll do]. By the end of this video, you'll have [end result]."

---

### Main Content (Main duration)

#### Section 1: [Topic Name] (X minutes)
- [ ] Explain concept
- [ ] Show visual/diagram
- [ ] Give real-world analogy
- [ ] Key takeaway

**Script:**
"[Explain concept]. Think of it like [analogy]. Let me show you [visual]. The key thing to remember is [takeaway]."

**Visuals:**
- [ ] Show diagram
- [ ] Highlight important parts
- [ ] Use arrows/annotations

---

#### Section 2: [Hands-on Coding] (X minutes)
- [ ] Open code editor
- [ ] Explain what you're doing
- [ ] Type code (don't copy-paste!)
- [ ] Explain each line
- [ ] Make intentional mistakes (then fix them)

**Script:**
"Now let's write the code. First, I'll [action]. Notice how I [detail]. This line does [explanation]. Oops, I made a typo - let me fix that. This is real coding, mistakes happen!"

**Code to Show:**
```python
# Show code here
# Explain each section
# Make it look natural
```

**Common Mistakes to Show:**
- [ ] Typo in variable name
- [ ] Missing import
- [ ] Wrong indentation
- [ ] Then fix it naturally

---

#### Section 3: [Testing/Demo] (X minutes)
- [ ] Run the code
- [ ] Show output
- [ ] Explain what you see
- [ ] Test edge cases
- [ ] Debug if needed

**Script:**
"Let's run this and see what happens. [Run command]. Great! As you can see, [explain output]. Now let's test with [edge case]. Hmm, that's interesting - let me check the logs."

**Hands-on:**
- [ ] Run command
- [ ] Show terminal output
- [ ] Open browser/UI
- [ ] Show results

---

### Summary (30 seconds)
- [ ] Recap what you did
- [ ] Key takeaways
- [ ] What's next

**Script:**
"So to summarize, we [recap]. The key things to remember are [takeaways]. In the next video, we'll [next topic]."

---

## Visual Checklist

- [ ] Screen recording software running
- [ ] Code editor visible and readable
- [ ] Terminal/console visible
- [ ] Cursor visible and moving naturally
- [ ] Zoom in on code when needed
- [ ] Use highlights/arrows for important parts
- [ ] Show browser/UI when relevant

---

## Audio Checklist

- [ ] Microphone quality good
- [ ] No background noise
- [ ] Speak clearly and at moderate pace
- [ ] Pause after important points
- [ ] Use natural, conversational tone
- [ ] Show enthusiasm!

---

## Editing Notes

- [ ] Remove long pauses (>3 seconds)
- [ ] Remove "umms" and "uhhs" (but keep some for natural feel)
- [ ] Add text overlays for key points
- [ ] Add arrows/highlights in post-production
- [ ] Add chapter markers
- [ ] Add intro/outro if desired

---

## Common Phrases to Use

**Starting:**
- "Let's get started!"
- "Here's what we're going to do..."
- "I'll show you step by step..."

**Explaining:**
- "The reason we do this is..."
- "This is important because..."
- "Notice how..."
- "Let me explain why..."

**Coding:**
- "I'll type this out so you can see..."
- "Let me add this line..."
- "Oops, I made a mistake - that's okay!"
- "Let me fix that..."

**Testing:**
- "Let's run this and see what happens..."
- "Perfect! As you can see..."
- "That's interesting, let me check..."

**Wrapping up:**
- "So to recap..."
- "The key takeaway is..."
- "In the next video, we'll..."

---

## Human Touch Elements

1. **Make Mistakes:**
   - Intentionally make typos
   - Show real debugging
   - "Oh wait, I need to fix this..."
   - Makes it feel authentic

2. **Personal Stories:**
   - "When I first learned this..."
   - "I remember struggling with..."
   - "A common mistake I see is..."

3. **Encouragement:**
   - "Don't worry if this seems complex..."
   - "You're doing great!"
   - "This is normal, we all go through this..."

4. **Real Examples:**
   - Use real-world scenarios
   - Show actual use cases
   - Connect to industry practices

---

## Example: Complete Script for "Writing the Producer Code - Part 1"

### Introduction (30 sec)
"Hey everyone! In the last video, we set up Confluent Cloud and created our Kafka topic. Now we're going to write the Python code that sends orders to Kafka. By the end of this video, you'll have a working producer that can send messages to your Kafka topic."

### Main Content

#### Section 1: Understanding What We Need (2 min)
"Before we write code, let's understand what we need. We need to:
1. Connect to Kafka
2. Generate order data
3. Send that data to the 'orders' topic

Think of it like sending a letter - we need the address (Kafka server), the letter (order data), and the mailbox (topic)."

[Show diagram of producer → Kafka]

#### Section 2: Setting Up the Project (3 min)
"Let's create our producer.py file. I'll open VS Code and create a new file in the producer folder."

[Open VS Code, create file]

"First, we need to import the libraries. I'll type this out:"

```python
from kafka import KafkaProducer
import os
import json
```

"kafka-python is the library that lets us talk to Kafka. os lets us read environment variables - we'll use this for our credentials. json is for formatting our data."

#### Section 3: Environment Variables (3 min)
"Now, we need to get our connection details. Instead of hardcoding them, we'll use environment variables. This is a best practice - never put secrets in code!"

```python
BOOTSTRAP = os.environ.get("KAFKA_BOOTSTRAP_SERVERS")
API_KEY = os.environ.get("CONFLUENT_API_KEY")
API_SECRET = os.environ.get("CONFLUENT_API_SECRET")
```

"These will come from our environment. Later, when we deploy, AWS will inject these automatically."

#### Section 4: Creating the Producer (5 min)
"Now for the fun part - creating the Kafka producer. This is where we connect to Confluent Cloud:"

```python
producer = KafkaProducer(
    bootstrap_servers=BOOTSTRAP,
    security_protocol="SASL_SSL",
    sasl_mechanism="PLAIN",
    sasl_plain_username=API_KEY,
    sasl_plain_password=API_SECRET,
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
)
```

"Let me explain each part:
- bootstrap_servers: This is our Confluent Cloud address
- security_protocol: SASL_SSL means secure, encrypted connection
- sasl_mechanism: PLAIN is the authentication method
- sasl_plain_username/password: Our API credentials
- value_serializer: Converts our Python dict to JSON bytes

Oops, I forgot the closing parenthesis - let me fix that. See? Real coding!"

[Fix the typo naturally]

#### Section 5: Testing the Connection (2 min)
"Let's test this. First, I need to set my environment variables. I'll do this in my terminal:"

[Show terminal]
```bash
export KAFKA_BOOTSTRAP_SERVERS="pkc-xxx.confluent.cloud:9092"
export CONFLUENT_API_KEY="your-key"
export CONFLUENT_API_SECRET="your-secret"
```

"Now let's run our script:"

```bash
python producer.py
```

[Run it, show it connecting]

"Great! It connected. You can see it says 'Connected to Kafka'. If you get an error, check your credentials - I've made that mistake many times!"

### Summary (30 sec)
"So we've created our Kafka producer connection. We imported the libraries, set up environment variables, and created a secure connection to Confluent Cloud. In the next video, we'll generate order data and actually send messages to Kafka. See you there!"

---

## Tips for Natural Delivery

1. **Don't Memorize:**
   - Use this as a guide, not a script
   - Speak naturally
   - It's okay to go off-script

2. **Show Your Process:**
   - Think out loud
   - "Hmm, what should I do here..."
   - "Let me check the documentation..."

3. **Engage with Students:**
   - "You might be wondering..."
   - "A common question I get is..."
   - "If you're stuck, pause here and..."

4. **Pace Yourself:**
   - Don't rush
   - Pause after important points
   - Let students absorb information

5. **Be Enthusiastic:**
   - Show you love what you're teaching
   - Get excited about breakthroughs
   - Celebrate when things work!

---

**Remember: The best Udemy courses feel like a friend teaching you, not a robot reading a script!**

