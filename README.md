# TripMate AI --- Multi-Agent Travel Planner

Planning a trip usually means jumping between different websites to
check flights, hotels, weather, and expenses. TripMate AI brings these
tasks together into one AI-powered travel planning assistant.

Instead of relying on a single LLM response, the application uses a
**multi-agent architecture** built with LangGraph. A supervisor decides
which specialist agents are needed, collects their results, and
coordinates the creation of a personalized travel itinerary.

The user can review the generated itinerary, approve it, or request
changes before receiving the final travel plan.

## What it does

-   **Flight assistance:** Provides airport and airline information,
    route guidance, estimated airfare context, and booking suggestions.
-   **Hotel discovery:** Searches the web for accommodation options
    relevant to the destination.
-   **Weather updates:** Retrieves current weather conditions and
    forecast information.
-   **Budget analysis:** Estimates travel expenses, highlights potential
    budget risks, and suggests ways to save money.
-   **Personalized itinerary:** Combines the available information into
    a structured, day-by-day travel plan.
-   **Human review:** Allows users to approve the draft itinerary or
    request revisions.
-   **Persistent workflow:** Uses PostgreSQL to save LangGraph execution
    state and resume interrupted workflows.

## How it works

TripMate AI follows a supervisor-based workflow:

1.  The user describes their travel requirements.
2.  An input guardrail checks whether the request is related to travel
    planning.
3.  The supervisor identifies the relevant specialist agents.
4.  Selected agents gather information and perform their respective
    tasks.
5.  The itinerary agent combines the results into a draft plan.
6.  The workflow pauses for human approval.
7.  Based on the user's approval or feedback, the final agent prepares
    the response.

### Architecture

``` mermaid
flowchart TD
    A[User Request] --> B[FastAPI Application]
    B --> C[Input Guardrail]
    C -->|Allowed| D[Supervisor Agent]
    C -->|Blocked| X[Guardrail Response]

    D --> E{Select Specialist Agents}

    E --> F[Flight Agent]
    E --> G[Hotel Agent]
    E --> H[Weather Agent]
    E --> I[Budget Agent]

    F --> J[Itinerary Agent]
    G --> J
    H --> J
    I --> J

    J --> K[Draft Travel Plan]
    K --> L[Human Approval]

    L -->|Approve or Request Changes| M[Final Agent]
    M --> N[Final Travel Plan]

    D -.-> O[(PostgreSQL Checkpointer)]
    K -.-> O
    L -.-> O
```

## The agents behind TripMate AI

  -----------------------------------------------------------------------
  Agent                               Responsibility
  ----------------------------------- -----------------------------------
  Supervisor Agent                    Understands the request and selects
                                      the required agents

  Input Guardrail                     Filters unrelated requests

  Flight Agent                        Retrieves airport and airline
                                      information and generates flight
                                      guidance

  Hotel Agent                         Searches for relevant accommodation
                                      information

  Weather Agent                       Retrieves current weather and
                                      forecast data

  Budget Agent                        Analyses estimated trip costs and
                                      budget feasibility

  Itinerary Agent                     Combines specialist outputs into a
                                      travel itinerary

  Human Approval                      Pauses the workflow for user review

  Final Agent                         Produces the final travel plan
                                      using the review feedback
  -----------------------------------------------------------------------

The supervisor does not need to activate every specialist for every
request. It routes the request according to the information required.

## Technology stack

  Technology                     How it is used
  ------------------------------ ------------------------------------------------
  Python                         Core application logic
  LangGraph                      Multi-agent orchestration and workflow control
  LangChain                      LLM integration
  Groq                           LLM inference
  Llama 3.3 70B Versatile        Language model
  Model Context Protocol (MCP)   Connects external tools and services
  Tavily                         Web search and hotel discovery
  AviationStack                  Airport and airline information
  OpenWeather API                Current weather and forecast
  FastAPI                        Backend API and application serving
  PostgreSQL                     Persistent workflow checkpoints
  Pydantic                       Request validation

## MCP integrations

TripMate AI uses MCP to connect its agents with external services.

-   **Tavily MCP:** Provides web search capabilities for accommodation
    discovery.
-   **AviationStack MCP:** Provides airport and airline information.
-   **Custom Weather MCP Server:** Exposes weather tools using the
    OpenWeather API.

The weather MCP server exposes two tools:

-   `get_current_weather(city)`
-   `get_forecast(city)`

Each MCP integration can be loaded independently, so a failure in one
external service does not automatically prevent unrelated integrations
from being used.

## Human-in-the-Loop

After generating a draft, LangGraph pauses execution and waits for the
user's decision.

The user can:

-   Approve the generated itinerary.
-   Reject it and provide feedback.
-   Ask the system to revise the travel plan.

The workflow then resumes using the saved execution state and generates
the final response.

## Database and persistence

PostgreSQL is used with LangGraph's `PostgresSaver` checkpointer.

This allows the application to preserve workflow state using a unique
thread ID. It is particularly useful when the workflow pauses for human
approval and needs to resume later.

## API endpoints

The FastAPI application exposes the following endpoints:

  -----------------------------------------------------------------------
  Method                  Endpoint                Purpose
  ----------------------- ----------------------- -----------------------
  GET                     `/`                     Serves the frontend

  GET                     `/health`               Checks API availability

  POST                    `/api/travel`           Starts a
                                                  travel-planning
                                                  workflow

  POST                    `/api/travel/approve`   Resumes the workflow
                                                  after user review
  -----------------------------------------------------------------------

### Example travel request

``` json
{
  "message": "Plan a 5-day trip from Bangalore to Goa within ₹25,000."
}
```

### Example approval request

``` json
{
  "thread_id": "your_existing_thread_id",
  "approved": false,
  "feedback": "Reduce hotel expenses and add more sightseeing options."
}
```

## Environment variables

Create a `.env` file in the project root and configure the following
variables:

``` env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATION_STACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key
DATABASE_URL=your_postgresql_connection_url
```

Keep your API keys and database credentials private. Never commit your
`.env` file to GitHub.

## Project structure

``` text
TripMate-AI/
├── app.py
├── backend.py
├── mcp_client.py
├── custom_weather_mcp_server.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── .env
├── templates/
│   └── index.html
└── static/
    ├── css/
    ├── js/
    └── assets/
```

The `templates/` and `static/` directories contain the frontend files
served by FastAPI. Adjust this tree if your actual frontend folders
differ.

## Current scope and limitations

TripMate AI is designed for **travel research, recommendations, and
itinerary planning**.

-   It does not directly purchase or book flight tickets.
-   Flight information and airfare guidance depend on the available
    AviationStack data and model-generated estimates.
-   Hotel discovery is based on web search; it does not guarantee room
    availability or reservation.
-   Budget figures are estimates, not confirmed quotations.
-   Weather information depends on the availability and accuracy of the
    external weather service.

Always verify prices, availability, and booking details with the
relevant provider before making a reservation.

## Why I built this

The goal behind TripMate AI was to explore how multiple AI agents can
collaborate on a practical problem rather than asking one language model
to handle everything.

Working on this project involved building a supervisor-driven workflow,
connecting external services through MCP, managing shared state with
LangGraph, and implementing a human approval step backed by PostgreSQL.

It is an ongoing project, and there is room to improve the accuracy of
travel data, reliability of external integrations, and overall user
experience.

------------------------------------------------------------------------

**TripMate AI --- Plan smarter. Travel better.**