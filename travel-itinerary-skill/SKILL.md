---
name: travel-itinerary-planner
description: Use when planning a multi-day trip and the user wants a full day-by-day itinerary document (not just a rough plan) — triggered by requests like "여행 일정 짜줘", "여행 계획 만들어줘", or when destination/dates/travelers for a trip are being discussed.
---

# Travel Itinerary Planner

## Overview

Turns loose trip details (destination, dates, travelers) into a single reference-quality markdown itinerary: verified logistics, a Google Maps link for every stop, walkable route links between same-day stops, restaurant picks near landmarks, and a prep checklist with real deadline dates. See `template.md` for the exact output skeleton.

## When to Use

- User asks to plan a trip, build a travel itinerary, or wants a day-by-day schedule
- Destination and trip dates are known (ask if missing)
- NOT for a single reservation lookup, a "이 동네에서 반나절 뭐하지" quick question, or picking one restaurant — those don't need the full template

## Process

**0. Trip style interview.** After confirming destination/dates/traveler count/accommodation, ask via AskUserQuestion (don't guess):
- 여행 성격 (관광 / 맛집·술집투어 / 휴양 / 액티비티, 복수선택 가능)
- 동행 유형 (커플 / 가족 / 친구 / 혼자)
- 체력·페이스 (빡빡하게 vs 여유롭게)
- 예산 감각 (가성비 vs 상관없음)

These answers set landmark density, late-night inclusion, and price tier for every later step.

**1. Verify time-sensitive facts.** WebSearch each of: business hours, regular closing days, seasonal events (e.g. flower bloom windows), admission fees, typical weather for the trip month. Never state these from memory — a wrong closing day or an out-of-season flower claim breaks the plan. Anything you can't confirm, write literally as "확인 필요" in the output — don't smooth it over into a confident-sounding guess.

**2. Generate Google Maps links.** Every place mentioned gets a search link. Consecutive same-day stops reachable on foot get a route link too. See Link Patterns below.

**3. Suggest restaurants near landmarks.** For meal times that land near a scheduled landmark, WebSearch 2-3 well-reviewed options within walking distance. State the routing reason for the pick (e.g. "역 건물 안 — 캐리어 이동 없음"), and flag whether it needs a reservation.

**4. Write to the fixed template.** Use `template.md` — don't improvise a different structure. This keeps every trip's document reusable and scannable the same way.

**5. Compute D-day deadlines.** For every prep item with a deadline relative to departure (pre-check-in forms, deposit charge dates, tour confirmation windows), calculate the actual calendar date from the trip's start date and put that date in the checklist — not a relative offset like "1주 전."

**6. Route sanity check.** Re-read each day's stop order. Flag anything that backtracks, has an unrealistic travel time between consecutive stops, or visits a place outside its open hours. Reorder and note why in one line if you change something.

## Google Maps Link Patterns

```
검색: https://www.google.com/maps/search/?api=1&query=<URL-encoded place name>
경로: https://www.google.com/maps/dir/?api=1&origin=<start>&destination=<end>&waypoints=<mid1>|<mid2>&travelmode=walking
```

Use the place's native-language name (with Korean gloss in the link text), not coordinates — coordinates don't render recognizably when clicked.

## Common Mistakes

- Stating business hours/season facts from training data instead of WebSearch — dates like flower bloom windows are exactly what goes stale
- Skipping the D-day math and writing "며칠 전" instead of an actual date
- Restaurant suggestions with no walking-distance or reservation note
- Skipping step 0's interview and guessing pace/budget from context instead
