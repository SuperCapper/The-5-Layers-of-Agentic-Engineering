# Implementing Structured Output with Local LLMs

For building LLM applications, local LLMs are an attractive option.

They allow us to keep our sensitive data and reduce our dependency on cloud APIs.

However, running the model locally is only the first step. In a practical application, the local LLM is usually part of a larger workflow. This means its responses often need to be consumed by another component.

In those situations, free-form text can be very difficult to work with. We want the output to follow some predictable structures.

That's exactly what Structured Output is for.

We can achieve that by first defining the expected shape, or schema, in advance. The local serving runtime then constrains the LLM generation to follow that schema. Finally, the LLM would give us a regular Python object that our code can easily parse.

In this post, we'll use Gemma 4 as our local LLM, Ollama as the serving runtime, and Pydantic to define and validate the output schema.

## 1. How Do We Implement Structured Output with a Local LLM?

### 1.1 A Smart-Home Case Study

Suppose we are building a smart-home application. The user asks a simple question:

Should the dishwasher run now or later?

Before answering, the application needs to extract device information, timing constraints, and electricity tariffs from household notes.

Since these notes contain private information, a local LLM is a natural fit as the first step. It can transform the original notes into a structured object that retains only the facts needed for scheduling while removing unnecessary personal details.

We can then pass this sanitized object to a more capable cloud LLM for reasoning and scheduling. Here, let's focus on the local transformation step.

The following is the household context we'll use:

```python
USER_QUESTION = "Should the dishwasher run now or later?"

SMART_HOME_CONTEXT = """
It is currently 18:30.

The activity log records that the robot vacuum completed today's kitchen pass
at 16:10 and returned to its dock. No more vacuuming is needed today.

The dishwasher's earliest start is 18:30. A cycle takes 90 minutes and uses about 1.2 kWh.
It must be complete before breakfast at 06:30. Because the dishwasher is beside
the bedrooms, it must stop running by 22:30.

The EV charger's earliest start is 18:30. Charging will take 120 minutes and use about
14 kWh. The car must be charged before its driver leaves at 07:00.

The dryer's earliest start is 19:00. Its cycle takes 75 minutes and uses about
3.2 kWh. It contains the football kit, which must be dry by 23:00. The dryer is
too loud later in the evening, so it must stop running by 21:30.

The washing machine's earliest start is 20:00. Its cycle takes 60 minutes and
uses about 0.9 kWh. It contains tomorrow's work clothes and must finish by 05:30.

A kitchen pass with the robot vacuum takes 45 minutes and uses about 0.2 kWh.
The vacuum's earliest start was 15:00.

The home energy controller permits only one flexible load to run at a time.
Electricity costs 0.45 per kWh from 17:00 to 20:00, 0.22 from 20:00 to 00:00,
0.12 from 00:00 to 06:00, and 0.25 from 06:00 to 17:00.
""".strip()
```

The goal of the local LLM is to retain the scheduling facts while leaving those personal details behind.

### 1.2 Define the Expected Structure

Next, we need to define what the sanitized object should look like.

The downstream component needs the current time, the device mentioned in the question, the controller capacity, and the electricity prices. It also needs the devices that still require scheduling, together with their runtime and timing requirements.

We can represent this using the following Pydantic models:

```python
from typing import Annotated

from pydantic import BaseModel, Field

ClockTime = Annotated[
    str,
    Field(
        min_length=5,
        max_length=5,
        description="Clock time in HH:MM format.",
    ),
]


class DeviceToSchedule(BaseModel):
    device_name: str
    duration_minutes: int
    energy_kwh: float
    earliest_start: ClockTime
    finish_by: ClockTime | None


class SchedulingContext(BaseModel):
    current_time: ClockTime
    focus_device: str
    max_concurrent_devices: int

    current_price_per_kwh: float
    off_peak_start: ClockTime
    off_peak_end: ClockTime
    off_peak_price_per_kwh: float

    devices_to_schedule: list[DeviceToSchedule] = Field(
        description=(
            "Devices that have not completed their work "
            "and still need to be scheduled."
        )
    )
```

Note that we have a nested schema, but the structure is relatively easy to follow. `SchedulingContext` contains the shared household facts and a list of `DeviceToSchedule` objects.

That's the shape we want the local LLM to output.

### 1.3 Setting Ollama and Local LLM

Before moving forward, make sure that Ollama is installed and running locally. You can install Ollama on Windows:

```
winget install Ollama.Ollama
```

On macOS or Linux, run:

```
curl -fsSL https://ollama.com/install.sh | sh
```

Once Ollama is installed, we can pull the Gemma 4 model:

```
ollama pull gemma4:e4b
```

We also need the Ollama Python client and Pydantic:

```
pip install ollama pydantic
```

Here, we use the compact 4B variant of the Gemma 4 model for our current case study.

### 1.4 Connect Pydantic to Ollama

Now, we connect the schema to our local model.

Here is how we can achieve that:

```python
import ollama

def call_local_llm(schema, instructions, prompt):
    response = ollama.chat(
        model="gemma4:e4b",
        messages=[
            {"role": "system", "content": instructions},
            {"role": "user", "content": prompt},
        ],
        think="medium",
        format=schema.model_json_schema(),
    )

    return schema.model_validate_json(response.message.content)
```

Two important things worth mentioning here:

- `model_json_schema()` converts our Pydantic model into the schema, and then passed into Ollama via the `format` argument.
- `model_validate_json()` parses the response into the same Pydantic model. This allows easy consumption in the downstream steps.

### 1.5 Make the Structured-Output Call

Now we can ask Gemma 4 to transform the household notes.

The instruction is simple:

```python
STRUCTURING_INSTRUCTIONS = """
Convert the supplied source material into the structured scheduling context.
Do not decide or propose a schedule.
""".strip()


def build_structuring_prompt(source_material):
    return f"""
User question:
{USER_QUESTION}

Source material:
{source_material}
""".strip()
```

Finally, we pass the complete `SchedulingContext` schema to the local model:

```python
one_step_context = call_local_llm(
    SchedulingContext,
    STRUCTURING_INSTRUCTIONS,
    build_structuring_prompt(SMART_HOME_CONTEXT),
)
```

That's it.

To achieve structured output with a local LLM, we first define the expected structure with Pydantic models, then use it to constrain the model generation, and finally parse the response back.

That's the mental model you need for structured output.

## 2. Valid Structure, Wrong Content

Now, let's put it into practice and see what Gemma 4 returns.

We run the one-step call and inspect the returned object:

```python
print(type(one_step_context).__name__)

print([
    device.device_name
    for device in one_step_context.devices_to_schedule
])
```

Here are the results:

```
SchedulingContext

[
    "Dishwasher",
    "EV Charger",
    "Washing Machine",
    "Robot Vacuum (Kitchen Pass)"
]
```

On the surface, everything seemed to work as expected.

Gemma 4 returned valid JSON that follows our schema. Pydantic also successfully parsed it into a `SchedulingContext` object.

However, there is one problem: the robot vacuum should not be included.

In the household notes, it clearly states that the robot vacuum completed its kitchen pass at 16:10 and that no more vacuuming is needed today. Therefore, it does not belong in `devices_to_schedule`.

This is an interesting result, and actually it gave us an important distinction in practice:

> Structured output only enforces the shape of the response. It does not, by itself, guarantee that the model puts the right information inside that shape.

So why did the model get it wrong?

In that one-step call, Gemma 4 has to perform several tasks at the same time, as required by the schema:

1. Determine which devices still need scheduling.
2. Extract the relevant facts for those devices.
3. Map those facts to the correct schema fields.
4. Assemble the final nested object.

That's a lot of work to do for such a small local LLM!

Therefore, as the schema becomes more complex, the model is under pressure to coordinate more decisions within a single generation, which makes it more likely to make mistakes.

So how can we address this challenge?

## 3. Decompose the Task

One practical way to address this challenge is to decompose the task.

In our current case study, instead of asking Gemma 4 to do everything in one go, we can break the task into two steps:

1. Determine which devices still need scheduling.
2. Extract the scheduling facts for those selected devices.

### Step 1: Determine the Scheduling Scope

For the first step, we need a new but small schema:

```python
class SchedulingScope(BaseModel):
    focus_device: str
    device_names_to_schedule: list[str]
```

We have the corresponding instruction:

```python
SCOPE_INSTRUCTIONS = """
Identify the focus device and the household devices that still need scheduling.
Do not decide or propose a schedule.
""".strip()
```

We pass the same user question and household notes to Gemma 4:

```python
scope = call_local_llm(
    SchedulingScope,
    SCOPE_INSTRUCTIONS,
    build_structuring_prompt(SMART_HOME_CONTEXT),
)

print(scope.model_dump_json(indent=2))
```

This time, the result is:

```json
{
  "focus_device": "Dishwasher",
  "device_names_to_schedule": [
    "Dishwasher",
    "EV charger",
    "Washing machine"
  ]
}
```

Note that the robot vacuum is no longer included. Gemma 4 correctly identifies that only the dishwasher, EV charger, and washing machine still need scheduling.

### Step 2: Fill the Final Schema

Next, we ask Gemma 4 to extract the remaining facts and fill the final `SchedulingContext`:

```python
DETAILS_INSTRUCTIONS = """
Convert the supplied source material into the structured scheduling context
for the supplied devices. Do not decide or propose a schedule.
""".strip()


details_prompt = f"""
Selected devices:
{json.dumps(scope.device_names_to_schedule)}

User question:
{USER_QUESTION}

Source material:
{SMART_HOME_CONTEXT}
""".strip()


decomposed_context = call_local_llm(
    SchedulingContext,
    DETAILS_INSTRUCTIONS,
    details_prompt,
)
```

We include the device list produced in the first step together with the original question and source material in the prompt above.

Now, let's inspect the complete result:

```python
print(decomposed_context.model_dump_json(indent=2))
```

This is what I obtained:

```json
{
  "current_time": "18:30",
  "focus_device": "Dishwasher",
  "max_concurrent_devices": 1,
  "current_price_per_kwh": 0.45,
  "off_peak_start": "00:00",
  "off_peak_end": "06:00",
  "off_peak_price_per_kwh": 0.12,
  "devices_to_schedule": [
    {
      "device_name": "Dishwasher",
      "duration_minutes": 90,
      "energy_kwh": 1.2,
      "earliest_start": "18:30",
      "finish_by": "06:30"
    },
    {
      "device_name": "EV charger",
      "duration_minutes": 120,
      "energy_kwh": 14.0,
      "earliest_start": "18:30",
      "finish_by": "07:00"
    },
    {
      "device_name": "Washing machine",
      "duration_minutes": 60,
      "energy_kwh": 0.9,
      "earliest_start": "20:00",
      "finish_by": "05:30"
    }
  ]
}
```

This time, the complete result is correct. The robot vacuum is excluded, all three device records match the original household notes, and the personal info is also gone.

Therefore, we can conclude that the direct approach returned a valid structure but with incorrect content, while our staged approach returned both a valid structure and correct content.

## 4. Final Thoughts

Local LLMs are an attractive option when an application works with sensitive data. With structured output, we can integrate local LLMs into a larger workflow, where the downstream components can easily consume LLMs' results.

In this post, we show that the implementation is straightforward. We start by defining the expected schema with Pydantic, then passing it to Ollama, and finally parsing the response back into a validated Python object.

In practice, however, one catch you should always keep in mind is that valid structure does not guarantee correct content. In our example, the direct call followed the schema but still produced the wrong results.

We effectively solved this problem by adopting a staged approach to separate scope determination from fact extraction, which led to correct results.

So, when a small local LLM struggles with a relatively complex schema, decomposition is one practical strategy worth trying.
