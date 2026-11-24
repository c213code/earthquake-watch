# Earthquake Watch - LangChain agent

## What problem it solves

Almaty is in a seismic zone. After an earthquake people want to know quickly:
**where was it, how strong was it, could I feel it, and what should I do?**

A normal chatbot can't answer this because it has no live data and often makes up numbers.
My agent gets **real earthquake data from USGS**, calculates distances, explains magnitudes
and makes an exact emergency kit list. It remembers the conversation, so you can ask follow-up questions.

## Tools

| Tool | Type | What it does |
|---|---|---|
| `get_city_coordinates(city)` | API (Open-Meteo Geocoding) | finds latitude and longitude of a city |
| `get_recent_earthquakes(latitude, longitude, days, radius_km, min_magnitude)` | API (USGS) | gets real earthquakes near a point, parses the JSON and calculates the distance to each one |
| `explain_magnitude(magnitude, distance_km)` | my logic | explains the magnitude and estimates if people can feel it at that distance |
| `make_emergency_kit(people, days)` | my logic (math) | emergency kit list with exact liters of water and number of meals |
| `save_note(text)` | file | saves a note to `my_notes.txt` |

Every tool has a docstring and argument types, and it returns an error message instead of crashing
(for example, if the city is not found or the API is down).

## LangChain parts

- **Prompts:** `ChatPromptTemplate` with a system prompt (role, rules, answer style) + user template `{input}`
- **LCEL chain:** `report_prompt | llm | PydanticOutputParser` -> returns an `EarthquakeReport` Pydantic object
- **Agent:** `create_tool_calling_agent` + `AgentExecutor`
- **Memory:** `RunnableWithMessageHistory` + `InMemoryChatMessageHistory` (one history per `session_id`)
- **LangSmith:** tracing is on (`LANGSMITH_TRACING=true`, project `earthquake-watch`)

## Model

**Claude Haiku 5.5** (`claude-haiku-5-5`) through `langchain-anthropic`.
I chose it because it is fast and cheap (about 10 times cheaper than Claude Opus), and it is still good at
tool calling (it picks the right tool and fills the arguments correctly) and follows the rules in the
system prompt, for example "never invent earthquakes". It also thinks step by step before it answers.

`AgentExecutor` and `create_tool_calling_agent` are in the `langchain-classic` package in the new LangChain versions,
so I import them from `langchain_classic.agents`.

## How to run

**Google Colab (easiest):**
1. Open `earthquake_watch.ipynb` in Colab.
2. Click the key icon (**Secrets**) on the left and add two secrets:
   - `ANTHROPIC_API_KEY` (from https://console.anthropic.com)
   - `LANGSMITH_API_KEY` (free, from https://smith.langchain.com)
3. Runtime -> Run all.

**On my computer:**
```bash
pip install -r requirements.txt
cp .env.example .env      # then put your keys in .env
jupyter notebook earthquake_watch.ipynb
```

API keys are never in the code. They are in Colab Secrets or in `.env`, and `.env` is in `.gitignore`.

## Example conversations

These are real answers from the notebook run (data from USGS on 24 Nov 2026).
The full output is in `earthquake_watch.ipynb`.

### Conversation 1 (follow-up questions, memory)

**User:** Were there any earthquakes near Almaty in the last 7 days?

*Tools used:* `get_city_coordinates(Almaty)` -> `get_recent_earthquakes(43.25, 76.91, days=7)`

**Agent:** Yes, but there were only **2 earthquakes**, and both were far from Almaty. I searched within 500 km for magnitude 2.5 and above.

| Magnitude | Location | Time (Almaty, UTC+5) | Depth | Distance from Almaty |
|---|---|---|---|---|
| **M4.9** | 39 km NW of Kuqa, China | 23 Nov, 07:14 (02:14 UTC) | 10 km | ~484 km |
| **M4.2** | 34 km NW of Kuqa, China | 23 Nov, 13:10 (08:10 UTC) | 10 km | ~491 km |

Both were near the far edge of the search area. Earthquakes this size that far away usually can't be felt in Almaty, and there were none close to the city.
**Tip:** Almaty is in an active earthquake zone, so make sure your heavy furniture and shelves are fastened to the walls.

**User:** What was the strongest one? Could people in Almaty feel it?

*Tools used:* `get_city_coordinates` -> `get_recent_earthquakes` -> `explain_magnitude(4.9, 484)`

**Agent:** The strongest one was the **M4.9** earthquake **39 km NW of Kuqa, China**, on **23 Nov at 07:14 Almaty time** (02:14 UTC). It was 10 km deep.
**People in Almaty probably didn't feel it.**
- An M4.9 earthquake is "light." People near the center feel it and objects shake, but damage is rare.
- It's usually felt up to about **150 km** away, and this one was about **484 km** from Almaty.
- This is only a rough estimate, because depth and soil type also matter.

**User:** Ok, I'm worried. Make an emergency kit list for my family of 4 people for 3 days and save it to my notes.

*Tools used:* `make_emergency_kit(people=4, days=3)` -> `save_note(...)`

**Agent:** It's normal to feel worried. Making a kit is a good way to get ready. Here's the list for **4 people for 3 days**, and it's **saved to my_notes.txt**:
- **Water: 36 liters** (3 L per person per day)
- **Food: 36 meals** that don't need cooking, like canned food, bars and nuts
- First aid kit and any medicines your family takes
- Flashlight, extra batteries and a power bank
- Copies of IDs and passports in a plastic bag
- Some cash
- Warm clothes and a blanket for each person
- A whistle to call for help

I also added the emergency number **112** to the note.

### Conversation 2 (another city)

**User:** Show me earthquakes of magnitude 4.5 or more near Tokyo in the last 14 days.

*Tools used:* `get_city_coordinates(Tokyo)` -> `get_recent_earthquakes(35.69, 139.69, days=14, min_magnitude=4.5)`

**Agent:** I found **8 earthquakes of magnitude 4.5 or higher** within 500 km of Tokyo in the last 14 days, from USGS data:

| Mag | Location | Time UTC (Tokyo) | Depth | Distance from Tokyo |
|---|---|---|---|---|
| **5.4** | 20 km NW of Anamizu | Nov 16, 04:24 (13:24) | 274 km | 323 km |
| **5.2** | 112 km SSE of Tateyama | Nov 21, 10:09 (19:09) | 10 km | 193 km |
| **5.2** | 86 km NW of Ryōtsu-minato | Nov 18, 09:37 (18:37) | 10 km | 373 km |
| 4.9 | 83 km ESE of Takahagi | Nov 10, 05:20 (14:20) | 35 km | 181 km |
| 4.6 | 68 km SSE of Tateyama | Nov 21, 17:30 (Nov 22, 02:30) | 65 km | 148 km |
| 4.6 | 130 km ENE of Hasaki | Nov 11, 03:56 (12:56) | 10 km | 230 km |
| 4.6 | 89 km SSE of Tateyama | Nov 10, 06:36 (15:36) | 10 km | 170 km |
| 4.5 | 2 km WNW of Hasaki | Nov 12, 13:27 (22:27) | 42 km | **101 km (closest)** |

All 8 were between magnitude 4.5 and 5.4. Earthquakes like these are common in Japan. Most were offshore or more than 100 km away, so people in Tokyo might have felt little or nothing.

### Conversation 3 (Russian + error handling)

**User:** Были ли землетрясения возле города Qwertyuiopcity?

*Tools used:* `get_city_coordinates(Qwertyuiopcity)` -> returned an error, the agent did not crash

**Agent:** Не удалось найти город **«Qwertyuiopcity»**: поиск координат не дал результата. Поэтому проверить землетрясения рядом с ним я пока не могу.
- **Проверьте написание.** Можно написать название по-русски или по-английски, например «Алматы» / «Almaty».
- **Укажите ближайший крупный город**, например Алматы, Шымкент, Талдыкорган или Бишкек.
- **Пришлите координаты** (широту и долготу). По ним я сразу проверю данные USGS.

**User:** Извини, я имел в виду Бишкек. Опасно ли землетрясение 5.5 на расстоянии 200 км?

*Tools used:* `get_city_coordinates(Bishkek)` -> `explain_magnitude(5.5, 200)` -> `get_recent_earthquakes(42.87, 74.59, days=7)`

**Agent:** Землетрясение **магнитудой 5.5 на расстоянии 200 км** для Бишкека обычно **не опасно**, но его, скорее всего, почувствуют:
- Это умеренное землетрясение. Рядом с эпицентром оно может повредить слабые здания.
- На расстоянии 200 км толчки обычно ощущаются (их чувствуют примерно до 300 км), но разрушения маловероятны.

**Землетрясения около Бишкека за последние 7 дней:** **M5.3**, 7 км к юго-востоку от Караканджи (Таджикистан), 18.11.2026 в 05:46 по времени Казахстана, **480 км от Бишкека**.
В экстренной ситуации звоните **112**.

## LangSmith traces

Project: `earthquake-watch` on https://smith.langchain.com

**One question from conversation 1.** The memory loads the history, then the agent calls Claude three times:
first it gets the coordinates, then the earthquakes, then it writes the final answer.

![Trace of conversation 1](screenshots/trace_1.png)

**Inside the `get_recent_earthquakes` tool call.** Claude passed the coordinates from the first tool,
and USGS returned a real M4.9 earthquake 484 km away. The agent used exactly this data in the answer.

![Tool call inside the agent](screenshots/trace_2.png)

**All runs of the project.** Every run finished without errors.

![All traces](screenshots/trace_list.png)
