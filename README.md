

import streamlit as st


def render_ai_assistant(hospital):

    st.title("Carepoint AI Assistant")
    st.write("Your hospital information assistant.")

    doctors = hospital.doctors

    st.subheader("🔍 Smart Doctor Search")

    query = st.text_input(
        "Search by doctor name or specialization"
    )

    if query:
        matches = doctors[
            doctors["name"].str.contains(
                query, case=False, na=False
            ) |
            doctors["specialty"].str.contains(
                query, case=False, na=False
            )
        ]

        if matches.empty:
            st.info("No matching doctors found.")
        else:
            st.dataframe(
                matches,
                use_container_width=True
            )

    st.divider()

    st.subheader("🤖 Carepoint Hospital Chatbot")

    if "carepoint_chat" not in st.session_state:
        st.session_state.carepoint_chat = []

    for msg in st.session_state.carepoint_chat:
        with st.chat_message(msg["role"]):
            st.write(msg["content"])

    question = st.chat_input(
        "Ask me about hospital services or doctors..."
    )

    if question:
        st.session_state.carepoint_chat.append({
            "role": "user",
            "content": question
        })

        q = question.lower()

        if "doctor" in q or "specialist" in q:
            answer = (
                "You can find doctors using our Smart "
                "Doctor Search above. Search by name "
                "or specialization."
            )

        elif "appointment" in q or "book" in q:
            answer = (
                "You can visit the Appointments section "
                "to manage your hospital appointments."
            )

        elif "bill" in q or "payment" in q:
            answer = (
                "Please visit the Billing section "
                "for billing and payment information."
            )

        elif "patient" in q:
            answer = (
                "You can manage patient information "
                "through the Patient Management section."
            )

        elif "hello" in q or "hi" in q:
            answer = (
                "Hello! Welcome to Carepoint Hospital. "
                "How can I help you today?"
            )

        else:
            answer = (
                "I can help you with doctors, "
                "appointments, patients and billing. "
                "What would you like to know?"
            )

        st.session_state.carepoint_chat.append({
            "role": "assistant",
            "content": answer
        })

        st.rerun()
