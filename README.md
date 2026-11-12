
import os
import streamlit as st
from openai import AzureOpenAI


def get_azure_client():
    endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    api_key = os.getenv("AZURE_OPENAI_API_KEY")
    api_version = os.getenv(
        "AZURE_OPENAI_API_VERSION", "2024-10-21"
    )

    if not endpoint or not api_key:
        return None

    return AzureOpenAI(
        azure_endpoint=endpoint,
        api_key=api_key,
        api_version=api_version
    )


def get_doctor_context(hospital):
    if hospital.doctors.empty:
        return "No doctor records are currently available."

    doctors = hospital.doctors[
        ["name", "specialty"]
    ].to_dict(orient="records")

    return "\n".join(
        f"Doctor: {d['name']}, Specialty: {d['specialty']}"
        for d in doctors
    )


def render_ai_assistant(hospital):

    st.title("Carepoint AI Assistant")
    st.caption(
        "Your hospital information and doctor-search assistant"
    )

    doctor_context = get_doctor_context(hospital)

    st.subheader("Find a Doctor")

    search_query = st.text_input(
        "Enter a doctor's name or specialization",
        placeholder="e.g. Cardiologist"
    )

    if search_query:
        matches = hospital.doctors[
            hospital.doctors["name"].str.contains(
                search_query, case=False, na=False
            )
            |
            hospital.doctors["specialty"].str.contains(
                search_query, case=False, na=False
            )
        ]

        if matches.empty:
            st.info("No matching doctors found.")
        else:
            st.dataframe(
                matches[["name", "specialty"]],
                hide_index=True,
                use_container_width=True
            )

    st.divider()
    st.subheader("Chat with Carepoint AI")

    client = get_azure_client()

    if client is None:
        st.warning(
            "Azure OpenAI is not configured yet. "
            "Doctor search is available; configure Azure "
            "OpenAI credentials to enable the AI chatbot."
        )

    if "carepoint_messages" not in st.session_state:
        st.session_state.carepoint_messages = []

    for message in st.session_state.carepoint_messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])

    prompt = st.chat_input(
        "Ask about hospital services or doctors..."
    )

    if prompt:
        with st.chat_message("user"):
            st.markdown(prompt)

        st.session_state.carepoint_messages.append(
            {"role": "user", "content": prompt}
        )

        if client is None:
            response = (
                "The AI chatbot is not configured yet. "
                "You can use the doctor search above while "
                "we connect Azure OpenAI."
            )
        else:
            try:
                messages = [
                    {
                        "role": "system",
                        "content": (
                            "You are Carepoint's hospital "
                            "information assistant. Answer "
                            "general questions about hospital "
                            "services and help users find doctors. "
                            "Use only this doctor directory:\n"
                            + doctor_context
                            + "\nDo not invent doctors, availability, "
                            "prices or hospital policies. Do not "
                            "diagnose, prescribe or recommend "
                            "medicines. For medical concerns, "
                            "recommend consulting a qualified "
                            "healthcare professional. Never request "
                            "or reveal private patient records."
                        )
                    }
                ]

                messages.extend(
                    st.session_state.carepoint_messages[-10:]
                )

                deployment = os.getenv(
                    "AZURE_OPENAI_DEPLOYMENT"
                )

                if not deployment:
                    raise ValueError(
                        "Azure OpenAI deployment name is missing."
                    )

                result = client.chat.completions.create(
                    model=deployment,
                    messages=messages,
                    temperature=0.3
                )

                response = result.choices[0].message.content

            except Exception as error:
                response = (
                    "Unable to contact the AI service. "
                    "Please check the Azure configuration. "
                    f"Details: {error}"
                )

        with st.chat_message("assistant"):
            st.markdown(response)

        st.session_state.carepoint_messages.append(
            {"role": "assistant", "content": response}
        )
