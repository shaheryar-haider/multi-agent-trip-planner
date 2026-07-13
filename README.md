# Multi-Agent Trip Planner

A sophisticated multi-agent AI system that generates budget-aware, weather-aware travel itineraries for trip planning.

## 🎯 Project Overview

This project demonstrates practical application of multi-agent systems by building a real-world trip planner for Paris. Three specialized agents collaborate iteratively to produce intelligent, constraint-aware travel plans:

1. **Itinerary Agent** (LLM-powered via OpenAI GPT-4o-mini)
   - Generates day-by-day itineraries
   - Integrates live weather forecasts
   - Outputs structured cost data for verification
   - Adapts based on critic feedback

2. **Budget Agent** (Rule-based)
   - Independently verifies total costs
   - Parses structured `ACTIVITY_COSTS_JSON` array
   - Returns deterministic verdicts: `WITHIN_BUDGET`, `OVER_BUDGET`, or `UNKNOWN`
   - Prevents LLM arithmetic errors from corrupting plans

3. **Critic Agent** (Rule-based)
   - Scores itineraries 0-10 (budget fit + activity variety)
   - Generates natural-language feedback
   - Drives iterative improvement loop
   - Stops when score > 8 and budget is met

## 🏗️ Architecture

```
Trip Request (destination, budget, preferences)
    ↓
Orchestrator (max 3 iterations)
    ↓
Itinerary Agent → [Weather Tool: Open-Meteo API]
    ↓
Budget Agent (verify costs independently)
    ↓
Critic Agent (score & generate feedback)
    ↓
If score > 8 AND budget OK: STOP
Else: Loop feedback back to Itinerary Agent
    ↓
Final Plan (best iteration across all runs)
```

## ✨ Key Design Decisions

### Structured Cost Verification
The Itinerary Agent outputs a machine-readable line at the end of every response:
```
ACTIVITY_COSTS_JSON: [0, 15, 12, 25, 19, 10, 0, 15, 10, 8]
```
This prevents LLM arithmetic errors from compromising the plan. The Budget Agent always sums independently from this structured field rather than trusting the LLM's prose.

### Iterative Refinement with Shared State
- A `TripState` data class maintains shared memory across all agents
- Critic feedback is appended directly to the next Itinerary Agent prompt
- The system tracks the best-scoring plan across all iterations (not just the last one)
- Full execution history is logged to JSON for debugging and review

### Weather Integration
- Live forecast data from Open-Meteo API is fetched before each iteration
- Influences activity scheduling and recommendations (e.g., staying hydrated during hot days)
- Falls back gracefully to mock forecast if API is unavailable

## 📊 Test Results

| Budget | Outcome | Iterations | Notes |
|--------|---------|-----------|-------|
| $600 (comfortable) | ✅ Passed iteration 1 | 1 | Included paid museums, bistro meals, snacks |
| $150 (tight) | ✅ Passed iteration 2 | 2 | Trimmed dining, kept museums & free attractions |
| $80 (very tight) | ✅ Passed iteration 2 | 2 | Stripped to free attractions & street food |

### Sample Interaction
```
--- Iteration 1 | ItineraryAgent ---
[Parsed activity costs: [0, 15, 12, 25, 19, 10, 0, 15, 10, 8] -> sums to $129.00]

--- Iteration 1 | BudgetAgent ---
verdict: OVER_BUDGET — $129.00 exceeds budget by $49.00.

--- Iteration 1 | CriticAgent ---
total_score: 5 — "Reduce cost to fit budget."

--- Iteration 2 | ItineraryAgent ---
[Parsed activity costs: [0, 8, 0, 15, 12, 7, 0, 15, 0, 6, 10] -> sums to $73.00]

--- Iteration 2 | BudgetAgent ---
verdict: WITHIN_BUDGET — $73.00 is within the $80.00 budget.

--- Iteration 2 | CriticAgent ---
total_score: 9 — "Looks good."
```

## 🚀 Getting Started

### Prerequisites
- Python 3.8+
- OpenAI API key
- pip

### Installation

1. Clone the repository:
```bash
git clone https://github.com/yourusername/multi-agent-trip-planner.git
cd multi-agent-trip-planner
```

2. Create a virtual environment (recommended):
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install dependencies:
```bash
pip install -r requirements.txt
```

4. Set up environment variables:
```bash
# Create a .env file in the project root
echo "OPENAI_API_KEY=your_api_key_here" > .env
```

### Running the Notebook
```bash
jupyter notebook Capstone_Project_By_Syed_Shaheryar_Haider.ipynb
```

Or with JupyterLab:
```bash
jupyter lab Capstone_Project_By_Syed_Shaheryar_Haider.ipynb
```

## 📝 Usage Example

```python
# The notebook demonstrates usage with:
trip_request = {
    "destination": "Paris",
    "duration_days": 4,
    "budget": 80,
    "preferences": ["museums", "cafes", "walking tours"]
}

# Run orchestrator with up to 3 iterations
final_plan = orchestrator.run(trip_request)
```

## 🔍 What Worked Well

1. **Cost Verification Independence**: The Budget Agent's independent verification caught cases where the LLM's stated total ($100) didn't match the parsed array ($75).

2. **Effective Feedback Loop**: The Itinerary Agent demonstrably changed plans in response to feedback, swapping expensive restaurants for street food and paid attractions for free walking areas.

3. **Real Weather Integration**: Live forecast data (high 30s Celsius for Paris) consistently appeared in recommendations for staying hydrated and scheduling indoor activities.

4. **Deterministic Auditing**: Budget verification uses plain Python arithmetic over regex-parsed JSON, ensuring every calculation is reproducible.

## ⚠️ Challenges & Limitations

1. **Non-monotonic Convergence**: The feedback loop doesn't guarantee improvement each iteration. One run saw costs move $129 → $92 → $99. Each revision is a fresh LLM generation, not constrained optimization.

2. **Keyword-Based Variety Scoring**: The Critic uses keyword matching (museum, hike, tour, market, etc.) rather than semantic understanding. Plans with novel but reasonable nouns ("Marais District," "Seine River walk") sometimes scored low despite being genuinely varied.

3. **Inconsistent LLM Arithmetic**: The Itinerary Agent got math correct in some runs and wrong in others, necessitating independent verification every time.

4. **No Weather-Driven Constraint**: Weather conditions currently provide information only; they don't directly constrain activity selection.

## 🔮 Future Improvements

- **Semantic Variety Scoring**: Use embeddings or LLM-based similarity to measure activity diversity beyond keyword matching.
- **Budget Trend Monitoring**: Track cost across iterations to prevent escalating overruns.
- **Weather-Driven Constraints**: Have the LLM skip certain activities entirely on poor-weather days, not just note them.
- **Retrieval Augmented Generation**: Store successful itineraries and use them as few-shot examples for faster convergence.
- **Multi-Agent Collaboration**: Introduce a "Local Expert" agent for domain-specific recommendations.

## 📊 Operational Considerations

### Guardrails
- Hard cap of 3 iterations prevents infinite loops
- Weather tool has explicit fallback to mock forecast if Open-Meteo API is unavailable

### Logging & Debugging
- Every agent's output recorded with iteration number and role
- Full execution history written to JSON file after each run
- Examples: `trip_plan_run_20240101_120000.json`

### Production Monitoring Metrics
If deployed continuously, monitor:
- **Iteration cap hit rate** (rising = feedback loop failing)
- **UNKNOWN verdict rate** from Budget Agent (signals LLM format drift)
- **Weather API failure rate**
- **OpenAI API latency & cost per call**

### Ethical & Safety Considerations
- Live weather data from Open-Meteo (falls back gracefully if unavailable)
- Itinerary details (cafes, prices) are AI-generated estimates, not independently verified
- Production system should clearly inform users that details may not be fully accurate
- Consider adding disclaimer about checking real prices and availability

## 📚 Dependencies

See `requirements.txt`:
- `openai-agents`: OpenAI Agents SDK for LLM orchestration
- `python-dotenv`: Environment variable management
- `requests`: HTTP client for Open-Meteo API calls



---

**Note**: This project was built as the capstone assignment for MGSC 695. Real-world production deployment would require additional safety checks, user verification of prices/availability, and compliance with terms of service for integrated APIs.
