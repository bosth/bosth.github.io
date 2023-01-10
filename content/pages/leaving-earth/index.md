---
title: "Leaving Earth is a Constraint Satisfaction Problem"
tags: [boardgames, csp, z3, python]
draft: true
---

In this article, I will demonstrate that Leaving Earth - or at least its mission-planning component - is one big constraint satisfaction problem. 

## Leaving Earth?

Leaving Earth is a independently-produced boardgame with lovely art and unique game play. In it, players recreate a space race similar to the one between the USSR and the USA that started in the 1950s. At the start of the game, a few missions will be revealed and the players to compete to be the first to complete each one, scoring points based on the complexity of each. Many of these missions are examples of actual human accomplishments - such as *Venus Lander* - while others - *Mars Station* - are more fanciful.

{{< figure src="mission_sample.webp" caption="Easy missions in Leaving Earth" >}}

{{< figure src="mission_list.webp" caption="In real life, the United States would have just edged the Soviet Union 43 to 42, with Germany scoring a single point." >}}

### Mission Planning

The core mechanic in Leaving Earth is planning the missions using a variety of rockets, capsules and other components to complete a series of maneuvers from one location to another. The key concept is that each maneuver has a difficulty - roughly approximating ∆v - and each rocket can produce a certain amount of thrust. The amount of thrust required to perform a maneuver is based on this simple formula: *thrust required = mass × difficulty*. The final factor, mass, is determined by the number of components aboard the spacecraft.

{{< mermaid >}}
graph TD
    A(Earth) -->|3| B(Suborbital Flight)
    B -->|5| C(Earth Orbit)
    A -->|8| C
    C -->|3| D(Lunar Orbit)
    D -->|2| E(Moon)
{{< /mermaid >}}

A simple mission plan for a spacecraft travelling from Earth to the Lunar surface might look like the following (the numbers in parentheses indicates the mass of that component):

| Maneuver                   | Difficulty | Rockets Used | Payload                        | Required Thrust |
|----------------------------|------------|--------------|--------------------------------|-----------------|
| Earth to Earth Orbit       | 8          | Saturn (20)  | Atlas (4), Juno (1), Probe (1) | 26 × 8          |
| Earth Orbit to Lunar Orbit | 3          | Atlas (4)    | Juno (1), Probe (1)            |  6 × 8          |
| Lunar Orbit to Moon        | 2          | Juno (1)     | Probe (1)                      |  2 × 2          |
 
As long as the rockets used in each stage of this mission can product the required thrust, then the maneuver can be completed. In the example above, a Saturn rocket produces 200 thrust, an Atlas 27 thrust and a Juno 4, so each maneuver succeeds.

An alternative route to space would be to use a two-stage launch to avoid the difficulty 8 maneuver required to go directly from the Earth's surface to Earth Orbit.

| Maneuver                         | Difficulty | Rockets Used | Payload                                   | Required Thrust |
|----------------------------------|------------|--------------|-------------------------------------------|-----------------|
| Earth to Suborbital Flight       | 3          | Soyuz (9)    | Soyuz (9), Atlas (4), Juno (1), Probe (1) | 24 × 3          |
| Suborbital Flight to Earth Orbit | 5          | Soyuz (9)    | Atlas (4), Juno (1), Probe (1)            | 15 × 5          |
| Earth Orbit to Lunar Orbit       | 3          | Atlas (4)    | Juno (1), Probe (1)                       |  6 × 8          |
| Lunar Orbit to Moon              | 2          | Juno (1)     | Probe (1)                                 |  2 × 2          |

A Soyuz produces only 80 thrust, but using it in a two-stage configuration means that it can still reach orbit. In fact, two Soyuz rockets can carry a payload of 7 mass into orbit, whereas a single Saturn can only carry 5. And since in the game's economy a Soyuz only costs $8 compared to a hefty $15 for a Saturn, the former starts to look like an attractive proposition. That is until you start planning, say, a mission to transport humans to Mars along with enough enough rockets to bring them up from the surface so they don't die, costing your space agency precious points.

There are many other mission planning complexities: Ion Drives, which are not expended but take time to travel from one location to another; slingshot maneuvers to travel to the outer reaches of the solar system; one-way manuevers; and more. But the basic concept I presented above is enough to move on.

# Constraint Satisfaction Problems

So how can Leaving Earth's mission planning be reduced to a Constraint Satisfaction Problem (CSP)?

Let's take the difficulty 5 maneuver from Earth to Suborbital Flight and rewrite it as a series of constraints. To simplify things, we'll assume that there are only Saturn and Soyuz rockets available.

{{< figure src="leavingearth_tabletop.jpg" caption="The Leaving Earth map" >}}

In the formula below ``saturn`` is the number of Saturn rockets fired in this maneuver and ``soyuz`` is the number of Soyuz rockets. The number ``1`` is the mass of the payload that we want to carry to space.

```
(saturn × 200) + (soyuz × 80) 
>= 
((saturn × 20) + (soyuz × 8) + 1) × 5
```

On the left size of the equation is the thrust that will be produced and on the right size we have the total mass of the spacecraft (including the rockets themselves and the payload) multiplied by the difficulty of the maneuver. Or in other words: *thrust required = mass × difficulty*.

This equation is quite easy to solve: any value of ``saturn`` or ``soyuz`` would satisfy this single constraint. But if the payload size increased from ``1`` to, say, ``5``, then the solutions are perhaps not so obvious.

```
(saturn × 200) + (soyuz × 80) 
>= 
((saturn × 20) + (soyuz × 8) + 5) × 5
```

## z3

This is where a CSP solver comes in handy. z3 is a toolkit for solving CSPs with bindings for several languages, including Python, which is the language I'll be using here.

z3 will help find the right number of rockets to fit the Leaving Earth equation *thrust required = mass × difficulty*. In other words the formula is our constraint and the number of rockets are the variables that need to be solved for.

### Solving a single maneuver

Rewriting the CSP above in z3 would look like this:

```python
from z3 import solve, Int

saturn = Int("saturn rockets")
soyuz = Int("soyuz rockets")

payload = 5
thrust = saturn*200 + soyuz*80
mass = saturn*20 + soyuz*9 + payload
difficulty = 5

solve(thrust >= mass * difficulty)
```

z3 will return the following result:

```python
[saturn rockets = 1, soyuz rockets = 0]
```

In other words, 1 Saturn rocket can transport a payload of 5 to Suborbital Flight. This certainly is *a* solution, but is it the optimal solution? For this, we have to decide what defines optimal. Is it the number of rockets? The total mass of the rockets? Or is it the *cost* of the rockets used? It's probably the cost.

Let's rewrite the function but with a cost optimization and let's add all the remaining rockets in as variables.

```python
from z3 import Optimize, Int

saturn = Int("saturn rockets")
soyuz = Int("soyuz rockets")
atlas = Int("atlas rockets")
juno = Int("juno rockets")

payload = 5
thrust = saturn*200 + soyuz*80 + atlas*27 + juno*4
mass = saturn*20 + soyuz*9 + atlas*4 + juno*1 + payload
difficulty = 5

cost = saturn*15 + soyuz*8 + atlas*5 + juno*1

solver = Optimize()
solver.add(thrust >= mass * difficulty)
solver.add(saturn >= 0)
solver.add(soyuz >= 0)
solver.add(atlas >= 0)
solver.add(juno >= 0)
solver.minimize(cost)

solver.check()
solver.model()
```

And this returns:

```python
[saturn rockets = 0, soyuz rockets = 1, atlas rockets = 0, juno rockets = 0]
```

And of course you can get a mix of rockets with the right conditions, such as by setting ``payload`` to ``21``:

```python
[saturn rockets = 1, soyuz rockets = 0, atlas rockets = 1, juno rockets = 0]
```

### Solving longer missions

When it's just a single maneuver, the constraints end up being quite simple; however, missions in Leaving Earth inevitably have multiple steps. Fortunately, this is not difficult to express in z3. Let's take the mission from Earth Orbit to the Moon. We have a maneuver of difficulty 3, followed by a maneuver of difficulty 2.

The key here is to express this mission *backwards*; in other words, we'll make sure we can travel from Lunar Orbit to the Moon first and then ensure we can travel from Earth Orbit to Lunar Orbit. And crucially, we'll increase the payload by the number of rockets expended during the trip from Lunar Orbit to the Moon because we had to carry those rockets from Earth Orbit!

| Stage | Maneuver                         | Difficulty | Payload                             |
|-------|----------------------------------|------------|-------------------------------------|
| 0     | Lunar Orbit to Moon              | 2          | 5                                   |
| 1     | Earth Orbit to Lunar Orbit       | 3          | 5 + mass of rockets used in Stage 0 |

In z3, this would be:

```python
from z3 import Optimize, IntVector, Int
solver = Optimize()
maneuvers = [2, 3] # reverse list of maneuvers to perform
payload = 5

saturn = IntVector("saturn rockets", len(maneuvers))
soyuz = IntVector("soyuz rockets", len(maneuvers))
atlas = IntVector("atlas rockets", len(maneuvers))
juno = IntVector("juno rockets", len(maneuvers))

cost = 0 # running total of the cost
for i, difficulty in enumerate(maneuvers):
    solver.add(saturn[i] >= 0)
    solver.add(soyuz[i] >= 0)
    solver.add(atlas[i] >= 0)
    solver.add(juno[i] >= 0)
    
    thrust = saturn[i]*200 + soyuz[i]*80 + atlas[i]*27 + juno[i]*4
    thruster_mass = saturn[i]*20 + soyuz[i]*9 + atlas[i]*4 + juno[i]*1
    mass = thruster_mass + payload
    solver.add(thrust >= mass * difficulty)

    cost += saturn[i]*15 + soyuz[i]*8 + atlas[i]*5 + juno[i]*1
    payload += thruster_mass

solver.minimize(cost)
solver.check()
solver.model()
```

Removing all the unused rockets, we are left with the following:

```python
[atlas rockets0 = 1, soyuz rockets1 = 1]
```

This is saying that for maneuver `0`, we need an Atlas and for maneuver `1`, we need a Soyuz. Remember that we planned this backwards, so `0` is the last maneuver (i.e. Lunar Orbit to the Moon) and `1` is the maneuver that came before it (i.e. Earth Obit to Lunar Orbit).

The amazing thing is that by simply changing the line ``maneuvers = [2, 3]``, we can solve any one-way mission in Leaving Earth that uses expendable rockets. 

Earth to the Moon? ``maneuvers = [2, 3, 8]``

Earth to the Moon with a two stage launch? ``maneuvers = [2, 3, 5, 3]``

Earth to Mercury with a two stage launch? ``maneuvers = [2, 2, 5, 3, 5, 3]``

### Ion Drives

Ion Drives work a little differently: they can't be used for maneuvers travelling to or from celestial bodies; they are not expended when they are used; they produce 5 thrust for every year the journey takes. As an example, if a spacecraft composed of a Probe (mass 1) and an Ion Drive (mass 1) travels from Earth Orbit to Lunar Orbit, then the journey will take 2 years to complete. The same formula - *thrust required = mass × difficulty* - applies, giving (1+1) × 3 = 6. Ion Drives generate 5 thrust per year so one year of travel would only provide 5 thrust and we need 6, hence two years.

Since Ions are not expended, they are very efficient as long as you aren't in a hurry - remember this a space *race*.

Let's add support for Ion Drives to the code above. First we need to add the Ion Drives' mass and cost *once* but add their thrust at each stage. Second, we need to introduce a new variable for the number of years each maneuver takes.

```
from z3 import Optimize, IntVector, Int
solver = Optimize()
maneuvers = [2, 5, 3] # reverse list of maneuvers to perform
payload = 5

saturn = IntVector("saturn rockets", len(maneuvers))
soyuz = IntVector("soyuz rockets", len(maneuvers))
atlas = IntVector("atlas rockets", len(maneuvers))
juno = IntVector("juno rockets", len(maneuvers))
ion = Int("ion drives")
year = IntVector("years travelled", len(maneuvers))
solver.add(ion >= 0)

cost = 0 # running total of the cost
years = 0 # running total of journey time

payload += ion * 1 # add the weight of the ion drives to the payload once
cost += ion * 10 # add the cost of the ion drives to the total once

for i, difficulty in enumerate(maneuvers):
    solver.add(saturn[i] >= 0)
    solver.add(soyuz[i] >= 0)
    solver.add(atlas[i] >= 0)
    solver.add(juno[i] >= 0)
    solver.add(year[i] >= 0)
    
    thrust = saturn[i]*200 + soyuz[i]*80 + atlas[i]*27 + juno[i]*4 + ion*year[i]*5
    thruster_mass = saturn[i]*20 + soyuz[i]*9 + atlas[i]*4 + juno[i]*1
    mass = thruster_mass + payload
    solver.add(thrust >= mass * difficulty)

    years += year[i]
    cost += saturn[i]*15 + soyuz[i]*8 + atlas[i]*5 + juno[i]*1
    payload += thruster_mass

solver.minimize(cost)
solver.check()
solver.model()
```

The code above is for a journey from Earth Orbit to Mercury Orbit. Because Ion Drives can't be used to reach space from Earth, the code would need to be more complex to support the full journey, but this is good enough for now.

| Stage | Maneuver                                 | Difficulty |
|-------|------------------------------------------|------------|
| 0     | Mercury Fly-by to Mercury Orbit          | 2          |
| 1     | Inner Planets Transfer to Mercury Fly-by | 5          |
| 2     | Earth Orbit to Inner Planets Transfer    | 3          |

```python
[ion drives = 1, 
 years travelled0 = 3,
 years travelled1 = 9,
 years travelled2 = 4]
```

For $10 in game money, a single Ion Drive will carry a payload all the way to Mercury! The only problem is that it will take 16 years ... but what if we ask z3 to optimize for the journey time?

```python
solver.minimize(cost)
solver.minimize(years)
```

Now cost is still the most important factor but years will also be minimized and the new results are that the journey will take 13 years. You can also flip the order of the two minimize statements so that the critical factor is the journey time and cost is only of secondary importance!

## Leaving Earth Solver

I put all of this and more together into a Python package to solve many (but not all) Leaving Earth missions, including rules from Outer Planets and Stations. It's a command-line tool that works like this:

```
Usage: les [OPTIONS] ORIGIN DESTINATION [PAYLOAD]

Options:
  --version                       Show the version and exit.
  -v, --verbose                   Verbose mode
  -j, --juno RANGE                Number of Juno rockets
  -a, --atlas RANGE               Number of Atlas rockets
  -s, --soyuz RANGE               Number of Soyuz rockets
  -p, --proton RANGE              Number of Proton rockets
  -n, --saturn RANGE              Number of Saturn rockets
  -i, --ion RANGE                 Number of Ion thrusters
  -c, --cost RANGE                Cost of mission
  -m, --minimize [time|cost|mass]
                                  Minimization goal
  --single-stage                  Check a single stage configuration for
                                  launches from Earth (by default only a two-
                                  stage configuration will be attempted)
  --aerobraking / --no-aerobraking
                                  Use aerobraking
  --rendezvous / --no-rendezvous  If rendezvous technology is available, Ion
                                  thrusters will be detached when no longer
                                  needed
  -t, --time RANGE                Number of time tokens
  -y, --year INTEGER RANGE        Year that journey starts  [1956<=x<=1986]
  --help                          Show this message and exit.
```

### Examples

To find the optimal solution for a trip taking 1 mass from Earth Orbit (``Eo``) to the Moon (``L``):

```
les Eo L
```

The results are a JSON description of the entire mission:

```json
{
    "components": {
        "juno": 5
    },
    "payload": 1,
    "mass": 5,
    "cost": 5,
    "time": 0,
    "plan": [
        {
            "origin": "Earth orbit",
            "destination": "Lunar fly-by",
            "difficulty": 1,
            "components": {
                "juno": 2
            },
            "thrust": 8
        },
        {
            "origin": "Lunar fly-by",
            "destination": "Lunar orbit",
            "difficulty": 2,
            "components": {
                "juno": 2
            },
            "thrust": 8
        },
        {
            "origin": "Lunar orbit",
            "destination": "Moon",
            "difficulty": 2,
            "components": {
                "juno": 1
            },
            "thrust": 4
        }
    ]
}
```

To find the optimal solution for a trip that taking 5 mass from Earth (``E``) to the Moon that uses no Saturn rockets, between 1 and 3 Soyuz rockets and where we are able to detatch the Ion Drive before landing on the Moon (this avoids moving unnecessary mass):

```
les E L 5 --saturn 0 --soyuz 1-3 --rendezvous
```

To find the optimal solution for the fastest trip carrying 10 mass from Earth Orbit to Jupiter Fly-by (``Jfb``) that leaves in 1960.

```
les Eo Jfb 10 --year 1960 --minimize time
```

Result:

```json
{
    "start": 1960,
    "end": 1963,
    "components": {
        "juno": 3,
        "soyuz": 1,
        "ion": 1,
        "atlas": 1
    },
    "payload": 10,
    "mass": 17,
    "cost": 26,
    "time": 3,
    "plan": [
        {
            "origin": "Earth orbit",
            "destination": "Inner planets transfer",
            "difficulty": 3,
            "components": {
                "soyuz": 1,
                "ion": 1
            },
            "aerobraking": false,
            "year": 1960,
            "time": 1,
            "thrust": 85,
            "slingshot": false
        },
        {
            "origin": "Inner planets transfer",
            "destination": "Venus fly-by",
            "difficulty": 2,
            "components": {
                "juno": 1,
                "atlas": 1,
                "ion": 1
            },
            "aerobraking": false,
            "time": 1,
            "year": 1961,
            "thrust": 36,
            "slingshot": false
        },
        {
            "origin": "Venus fly-by",
            "destination": "Jupiter fly-by",
            "difficulty": 1,
            "components": {
                "juno": 2,
                "ion": 1
            },
            "aerobraking": false,
            "year": 1962,
            "time": 1,
            "thrust": 13,
            "slingshot": true
        }
    ]
}
```

### How it works

The core concept is identical to what I described above but ``les`` also finds the optimal paths from the origin to the destination. In most cases there are multiple paths between two points and the optimal path depends on a number of external factors. Even travelling to the Moon from Earth presents many options:

{{< mermaid >}}
graph TD
    E((Earth)) -->|3| ESo(Suborbital Flight)
    ESo -->|5| Eo(Earth Orbit)
    E   -->|8| Eo
    Eo  -->|3| Lo(Lunar Orbit)
    Lo  -->|2| L((fa:fa-ban Moon))
    Eo  -->|1| Lfb(Lunar Fly-by)
    Lfb -->|2| Lo
    Lfb -->|4| L
{{< /mermaid >}}

By default, ``les`` will try to predict a few plausible paths and solve for each of those, selecting the best solution at the end.

``les`` is aware of which maneuvers support Ion Drives and can detatch these opportunistically when they will no longer be needed to complete a mission. Conversely, it will carry unused Ions from one location to the next if they will be used at a future stage.

It also can make use of aerobraking, slingshot maneuvers and Proton rockets from the expansions. It doesn't support the resuable shuttles, however, and it won't increase or decrease the difficulty of maneuvers to change travel times. The biggest missing feature, howver, is probably not being able to plan return missions (e.g. going to the Moon and back again).


{{< button href="https://github.com/bosth/les/" target="_blank" >}}
Source code
{{< /button >}}
