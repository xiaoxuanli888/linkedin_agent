import os
from openai import OpenAI
import streamlit as st

# Initialize OpenAI client (expects OPENAI_API_KEY in environment)
client = OpenAI()

# ---- 1. Your base drafts (style) ----
BASE_DRAFTS = {
    "connection": """Hi [Name], I’ve been following [Company]’s journey—congrats on [milestone/recent achievement]. Your path from [previous area/field] to your current role at [Company] is really impressive. Would love to connect.""",

    "peer_first": """Hi [Name],
how are you today?

I’ve been a long-time lurker — I came across your work through [how you found them] and started following what you share. It’s basically been a free masterclass, so thank you for putting it out there.

I’m reaching out because of [company/team/role]. From what I understand, you’re looking for someone who can identify and test new growth levers, collaborate cross-functionally to launch experiments, and create content that educates your audience. I’ve led growth experiments and lifecycle initiatives before, and I’m actively deepening my work in [relevant area].

If you’re open to it, I’d love a quick 15–20 minute call to better understand what you’re aiming to achieve with this role and what “great” looks like on your team.

Thanks again for sharing your knowledge so openly — it truly makes an impact.

Best,
Li""",

    "peer_followup": """Hi [Name],

how are you doing? Just wanted to gently follow up in case my message got buried. I reached out because you’re working in [role/team/company], and I was hoping to learn more about what the team is looking for in this role. I’ve already sent in my application.

I recently moved to [city], and you seem like a genuinely cool person I’d love to connect with regardless of what happens with this role.

Wish you a good day.

Best regards,
Li""",

    "hiring_manager": """Hi [Name], thanks for connecting!

I’m really impressed by your experience in [field] and how you’ve managed to succeed across different countries and contexts. It’s also nice to see that you’re based in [city] and supporting [Company] in a remote or international setup.

I’m reaching out because I saw your post about the [Role Title] position and I’ve already applied. From what I understand, the team is looking for someone who can bring structure to marketing operations — owning website and landing page updates end to end, coordinating across marketing, design, and development, and continuously improving internal processes and documentation. I’ve done all of that in my previous roles, including at [Previous Company 1] and [Previous Company 2].

I recently moved to [city] to explore opportunities and life here. Would you be open to a quick 15–20 minute call after the new year? I’d love to hear more about what you’re looking for in this role and share a bit about how I’ve approached similar challenges in the past.

Best regards,
Li"""
}

# ---- 2. Prompt templates (agent instructions) ----
MESSAGE_TEMPLATES = {
    "connection": """
You are helping me write a short, personalized CONNECTION MESSAGE on LinkedIn.

Tone:
- Similar style to the base draft
- Warm, concise, specific, not salesy
- Natural and human

Base draft of my usual message:
---
{base_draft}
---

Using this draft as a STYLE REFERENCE and the LinkedIn profile, write a message that:
- Mentions the person by first name
- References their company and/or a recent achievement if visible
- Briefly compliments their background or path
- Ends with a simple, low-pressure connection line (e.g. "Would love to connect.")

My details (if relevant):
Name: {my_name}
Role: {my_role}
Location: {my_location}
Target role / goal: {my_goal}

LinkedIn profile text:
---
{profile}
---

Constraints:
- 40–90 words
- Do NOT invent very specific facts that are not in the profile.
- Keep it simple and easy to paste into a connection request.

Return ONLY the final message text, nothing else.
""",

    "peer_first": """
You are helping me write a FIRST MESSAGE to a PEER on LinkedIn (someone in a similar or adjacent area).

Tone:
- Exactly like the base draft: warm, thoughtful, specific, slightly long-form
- Friendly but professional
- Curious and appreciative

Base draft of my usual message:
---
{base_draft}
---

Using this draft as a STYLE REFERENCE and the LinkedIn profile, write a message that:
- Starts with a greeting and a short "how are you" style opener
- Mentions how I came across their work or profile (based on the profile text)
- Acknowledges something I genuinely appreciate about their work or journey
- Briefly explains why I'm reaching out (role/team/company/area of interest)
- Mentions that I have relevant experience (based on my details and profile)
- Asks for a short 15–20 minute call to learn more

My details:
Name: {my_name}
Role: {my_role}
Location: {my_location}
Target role / goal: {my_goal}

LinkedIn profile text:
---
{profile}
---

Constraints:
- 140–220 words
- Maintain the overall structure and feel of the base draft
- Do NOT make up precise metrics or projects that aren't in the profile

Return ONLY the final message text.
""",

    "peer_followup": """
You are helping me write a FOLLOW-UP MESSAGE to a PEER on LinkedIn (I already sent a first message and haven't heard back).

Tone:
- Gentle, respectful, low-pressure
- Similar style to the base draft: friendly, human, slightly informal
- Shorter than the first message

Base draft of my usual follow-up message:
---
{base_draft}
---

Using this as a STYLE REFERENCE and the LinkedIn profile, write a message that:
- Starts with a greeting and a light "how are you" tone
- Gently acknowledges this is a follow-up in case the previous message got buried
- Briefly reminds them why I reached out (role/team/company/area)
- Mentions that I’ve applied / am interested but keeps it human, not pushy
- Adds a human detail (e.g. that I moved to their city / would like to connect regardless of the role)

My details:
Name: {my_name}
Role: {my_role}
Location: {my_location}
Target role / goal: {my_goal}

LinkedIn profile text:
---
{profile}
---

Constraints:
- 80–160 words
- Keep it very friendly and easy to say yes/no to
- Do NOT pressure them

Return ONLY the final message text.
""",

    "hiring_manager": """
You are helping me write a MESSAGE TO A HIRING MANAGER or decision maker on LinkedIn.

Tone:
- Similar to the base draft: confident but warm, structured, specific
- Professional, not spammy
- Respectful of their time

Base draft of my usual message:
---
{base_draft}
---

Using this as a STYLE REFERENCE and the LinkedIn profile, write a message that:
- Thanks them for connecting (if relevant)
- Briefly acknowledges their experience or role (based on their profile)
- Clearly states that I saw a role (or type of role) and have applied / am interested
- Summarizes what I understand the role/team needs in 1–2 sentences
- Connects that to my background and experience (based on my details)
- Politely asks if they’d be open to a short 15–20 minute call

My details:
Name: {my_name}
Role: {my_role}
Location: {my_location}
Target role / goal: {my_goal}

LinkedIn profile text:
---
{profile}
---

Constraints:
- 140–230 words
- Maintain the structure and vibe of the base draft
- Do NOT invent fake details; keep experience descriptions general if needed

Return ONLY the final message text.
"""
}

# ---- 3. Core AI function ----
def generate_message(
    profile_text: str,
    message_type: str,
    my_name: str = "",
    my_role: str = "",
    my_location: str = "",
    my_goal: str = ""
) -> str:
    base_draft = BASE_DRAFTS[message_type]
    template = MESSAGE_TEMPLATES[message_type]

    prompt = template.format(
        base_draft=base_draft,
        my_name=my_name or "Li",
        my_role=my_role or "Marketing / Growth professional",
        my_location=my_location or "Berlin",
        my_goal=my_goal or "exploring growth and marketing roles in tech/startups",
        profile=profile_text.strip()
    )

    response = client.chat.completions.create(
        model="gpt-4.1-mini",  # you can change this to another chat model if you like
        messages=[
            {
                "role": "system",
                "content": "You write highly personalized, natural LinkedIn messages in the user's style."
            },
            {
                "role": "user",
                "content": prompt
            }
        ],
        temperature=0.7,
    )

    return response.choices[0].message.content.strip()

# ---- 4. Streamlit UI ----
st.set_page_config(page_title="LinkedIn Message Agent", page_icon="💌")

st.title("💌 LinkedIn Message Agent")
st.write("Paste a LinkedIn profile, choose a message type, and get a ready-to-send message in your style.")

with st.sidebar:
    st.header("Your details")
    my_name = st.text_input("Your name", value="Li")
    my_role = st.text_input("Your current role / headline", value="")
    my_location = st.text_input("Your location", value="Berlin")
    my_goal = st.text_area(
        "Your target role / goal (optional)",
        placeholder="e.g. exploring growth generalist roles in AI / SaaS startups"
    )

st.subheader("1. Paste LinkedIn profile text")
profile_text = st.text_area(
    "Copy the relevant parts: headline, About, current role, maybe 1–2 past roles.",
    height=260,
    placeholder="e.g. About section, current position, a few bullet points..."
)

st.subheader("2. Choose message type")
label_to_type = {
    "Connection message": "connection",
    "Peer – first message": "peer_first",
    "Peer – follow-up": "peer_followup",
    "Hiring manager message": "hiring_manager"
}
message_label = st.selectbox(
    "What kind of message do you want to generate?",
    list(label_to_type.keys())
)
message_type = label_to_type[message_label]

if st.button("✨ Generate message", type="primary"):
    if not profile_text.strip():
        st.warning("Please paste some LinkedIn profile text first.")
    else:
        with st.spinner("Generating your personalized message..."):
            try:
                msg = generate_message(
                    profile_text=profile_text,
                    message_type=message_type,
                    my_name=my_name,
                    my_role=my_role,
                    my_location=my_location,
                    my_goal=my_goal
                )
                st.success("Done! Here's your message:")
                st.code(msg, language="markdown")
                st.markdown("You can now copy & paste this directly into LinkedIn 💬")
            except Exception as e:
                st.error(f"Something went wrong: {e}")
