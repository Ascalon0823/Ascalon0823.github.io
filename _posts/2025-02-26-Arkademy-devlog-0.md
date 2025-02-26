---
layout: single
title: Arkademy Devlog 0
date: 2025-02-26 08:12 +0800
categories: Personal
tags: ["Devlog","Arkademy"]
toc: true
toc_lable: "Contents"
---

# Introduction
---
**Arkademy** is my personal side project ~~that has been restarted and reprototyped for million times~~, and this is the first dev log for the project. Hope that I do not have to restart a new one...😭

This dev log is intented to brief the plan I had for **Arkademy**. If you want to try the current release now, you can go [here](/Arkademy/index.html)


# The Game
---
**Arkademy** is a 2D pixel action online dungeon crawler. Players can build their characters using the rewards from finishing a run and then start harder and more rewarding ones

| Context | Description |
|---------|-------------|
| Graphic | 2D Pixel    |
| Genre   | Top-down Action |
| Platform | Web        |
| Control | Touch |
| Network | Server-Client |
| No. of Players | 1 |

## Game Loop
The main game loop is divided into two maps, **Campus** and **Rift**. 

**Campus** is basically the lobby in similar games where the player can purchase items and build characters freely.

**Rift** is where the player needs to control their character to fight agains npc enemies in a procedurally generated map.

**Rift** ----kill enemies----> **Campus** ----build characters----> **Rift**

## Mechanism
### Rift Progress

During a Rift run, the player need to defeat the generated enemies as fast as possible to gain progress for the Rift. There will be a fixed time limit for each Rift run, the player can only **pass** a Rift run by filling the progress before time limit.

### Rift Difficulty

Rift Difficulty is a number indicates the difficulty of a Rift run. This number affects the strength of enemies generated in the run, as well as the reward when the player passes the run.
New difficulty will be unlocked once the player passes a run.

### Rift Rewards

Upon passing a Rift run, the game will generate random rewards for the player like currency, material, and equipments. 

### Failed Rift

If the time run out before the player can fill the Rift progress, the run is considered as **failed**. The player can still continue the run and filling the progress. However, new difficulty will not be unlocked and majority of the rewards will not be rewarded after complete the Rift.

### Character Building

The player can level up abilities of their character and purchase or enhance equipments in the Campus by spending currencies.


# Architecture
---
## Client
The client will be developed using Unity WebGL support. Since it is a single page web app, it will be temporarily hosted on this site.

Currently it is at prototype stage and can be tested from [here](/Arkademy/index.html). Since the server is not running yet, the player data is handled using [Unity`s WebGL PlayerPref system](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/PlayerPrefs.html) temporarily, which will be lost if clear or change browser.

## Server
The server will be developed using Golang GIN + MongoDB and self hosted during development. I might also find some free hosting plans to host the test environment for the public to test.

Currently it is blank and I will be building it from scratch. Originally my plan is to use Firebase since it handles most backend functionalities for me, but some how it [only supports mobile platform for production](https://firebase.google.com/docs/unity/setup#desktop-workflow), why...?


# Tasks
---
At current stage these are the tasks I have completed and need to be done.

- [x] Client build and deploy flow
- [x] 2D Game basics (Control, character, camera)
- [x] Basic game loop
- [ ] Rift rewards (Currencies, Equipments) 
- [ ] Character building (Level up, Shops)

- [ ] DB setup
- [ ] User register and authentication
- [ ] Rift reward generator API
- [ ] Character building API

Some additional tasks I am considering
- [ ] Multiplayer using relay and Mirror I guess?
- [ ] Admin portal using Vue
- [ ] User portal using Vue