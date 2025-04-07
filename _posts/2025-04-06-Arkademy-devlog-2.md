---
layout: single
title: Arkademy Devlog 2
date: 2025-04-06 16:58 +0800
categories: Personal
tags: ["Devlog","Arkademy"]
toc: true
toc_lable: "Contents"
---

# Updates
---
- Hosting strategy
- Http client in Unity
- Game play

# Backend
---
There are multiple problems due to browsers` security practices to stop me from hosting my webgl front end on github pages. I will explain one by one chronologically.


### 1. Github size limitation

This one is simple. Unity builds a wasm object and need to be pushed to github repo. This single file can easily exceed 50MB limitation of github repo. 

I attempted to use Large File Storage support by Github, but apparently it will be excluded from Github Page build action, thus it will not appear in the Page at all.

For now the only solution is to compress the contents in build setting. In the `Player Settings > Player > Webgl > Publishing settings`, choose `Gzip` as compression format creates wasm that is around 12MB. 

Eventually as the assets grows, it will inevitably exceeds the limit even with compression. By then I will need to find a new way to host the contents.

### 2. CORS limitation

The first error I encountered after I connected the game to backend is this error, which does not happen in editor and local testing since they are on the same domain as my local server.

The idea is that, by default webpage from domain A can not grab resource from domain B. To allow that, changes need to be made from the server of domain B to allow request from different domain.

In my case, the hosted webgl is on `github.com` domain and my server is hosted on local host, which only accepts request from local host.

To enable CORS, I just need to add a middleware to my gin server.

``` go
func CORSMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		origin := c.Request.Header.Get("Origin")
		c.Writer.Header().Set("Access-Control-Allow-Origin", origin)
		c.Writer.Header().Set("Access-Control-Allow-Credentials", "true")
		c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization, accept, origin, Cache-Control, X-Requested-With")
		c.Writer.Header().Set("Access-Control-Allow-Methods", "POST, OPTIONS, GET, PUT, PATCH")

		if c.Request.Method == "OPTIONS" {
			c.AbortWithStatus(204)
			return
		}

		c.Next()
	}
}
```

Also, since I use `PATCH` method, I need to handle preflight request.

Notice that requests are accepted from any origins which is not a good idea, should changes this when I have a proper hosting.

### 3. HTTPS to HTTP

After handling the CORS, preflight and CORS related errors are gone. However I was still not able to make request to my server as I encountered Mixed Content error. This is caused by my hosting setup again, as my webgl is running on `github.com` domain which is **HTTPS** and my server running on local host which is **HTTP**.

Unfortunately unlike CORS there is no easy fix as I need to either host my server code on some hosting service provider, or make the local server running on HTTPS. 

This is where I decided to change my hosting strategy for now since I want to work on the gameplay already. Currently I host the webgl on both the github page and my local mac, the github page version will run in offline mode without accessing the server, and the local mac will run 'online' mode together with the local server. 

Both version should behave the same in game play, the only difference is the online mode will save the data in database and offline mode will save in the browser.

In the future, I will host both the webgl and server on some platform, since the server is already running in docker, it should be easier to transit to cloud.

# HTTP client in unity webgl

As mentioned in the previous devlog, the HTTP Client implementation using `System.Net` does not work on Webgl. One alternative is to use the BestHttp unity asset since I used it before, however it is not free now. So I checked to see if Unity provides any native api and found that `UnityWebRequest` class exists.

After playing with it, I find that it fulfills all my needs for now although it is not as well structured as the .Net one. 

``` c#
public async Task<T> Get<T>(string url)
{
    var uri = new Uri(BaseUrl, new Uri(url,UriKind.Relative));
    var request = UnityWebRequest.Get(uri);
    request.SetRequestHeader("Authorization", Token);
    await request.SendWebRequest();
    var result = request.result;
    if (result != UnityWebRequest.Result.Success)
    {
        return default;
    }
    return JsonConvert.DeserializeObject<T>(request.downloadHandler.text);
}

public async Task<UnityWebRequest> Post(string url, string payload)
{
    var request = UnityWebRequest.Post(new Uri(BaseUrl, new Uri(url,UriKind.Relative)), payload, "application/json");
    request.SetRequestHeader("Authorization", Token);
    await request.SendWebRequest();
    return request;
}

public async Task<UnityWebRequest> Patch(string url, string payload)
{
    var request = UnityWebRequest.Post(new Uri(BaseUrl, new Uri(url,UriKind.Relative)), payload, "application/json");
    request.method = "PATCH";
    request.SetRequestHeader("Authorization", Token);
    await request.SendWebRequest();
    return request;
    
}

```

Since my backend service code is pretty simple, changing the api calls are pretty fast, and now my http requests works.

Additionally, I need to implement a way to check the current hosting url and decide if it should use online or offline mode. Unity provides the `Application.absoluteURL` function for that.

``` c#
public static bool Offline
{
    get
    {
        if (!Application.isEditor)
        {
            return !Application.absoluteURL.Contains("192.168");
        }
        return Env().offline;
    }
}
```

By doing so, I can use one build to handle both mode. Also in editor I use a simple scriptable object to decide the environment information for the backend service, which make debug easy.

# Game play

### 1. The Character Building

The character building is currently consist of two part.

- **XP investment**

Spend XP obtained from rift reward on character Attributes like life, attack to increase the stats. The amount of XP required to level up a attribute is different, like 100 XP for 5 HP.

The amount of invested XP on each attribute are saved with character record, and the stats return from investment is calculate on runtime.

This implementation has a few benefit.

- Single source of truth, since only the amount of XP is saved instead of saving both XP and return. Thus balancing the return of stats just need to change the profile and forget about updating the return of the existing saves. Also during runtime, it does not have to update the return when investment amount changed.

- Variety for different character, as each race can have different investment profile. For example, 100 XP can give 5 HP for human, but for elf it needs 200 XP for 5 HP.

The current calculation simply consist of 

- initial cost per level
- additional cost per level
- return rate

To get current level based on invested XP:

``` c#

Mathf.FloorToInt((float)(Math.Sqrt(initialCost * initialCost + 8 * investedXP * additionalCost) - additionalCost) / additionalCost / 2);

```

- **Shop**

Currently it is only implemented with a simple UI. Since the equipment system is still not finished yet, the actual functionality of shop needs to wait.

### 2. Ability

One of the most difficult part for me is implementing a scalable yet clean ability system.

Scalable means that I can create any ability effects, and clean means I don't have to write extra code for new ability effects.

For now I have break the ability into two parts, one is ability execution, another one is ability effect.

The execution is fairly simple since there are only few ways to use an ability.

- **Passive**, which do stuff automatically, on mounted/every interval/triggered/every update.
- **One time**, which do stuff instantly one time when command comes.
- **Continuous**, which do start doing stuff when command comes and stop when command ask to cancel.


Thus I can just create three MonoBehaviour to handle each type, and the common data like cool down, use time can be represented using a scriptable object for each ability.

The difficulty lies in using data to represent an effect. And one big problem is Unity editor does not handle serialization of inherited class.

A few methods I have investigated:

- **Enum of effects type in giant Effect class**

``` c#

public class Ability : ScriptableObject{
    //...
    public List<Effect> effects;

}
public class Effect{
    public enum Type{
        Damage,
        Heal,
        Dot
        // etc...
    }
    public Type type;
    public int damageAmount;
    public int healAmount;
    public void Apply(){
        switch(type){
            case Damage:
                DoDamage();
                break;
            case Heal:
                DoHeal();
                break;
            //...
        }
    }
}

```
Pros: 
- At least you can turn the type into data and represent it easily

Cons: 
- Giant switch case to handle actual logic, giant class and list of data 


``` c#

public class DamageEffect : Effect{
    public int damageAmount;
    public override void Apply(){
        // Do damage
    }
}

public class HealEffect : Effect{
    public int healAmount;
    public override void Apply(){
        // Do heal
    }
}
```

- **Inheritance**

Pros: 
- Clean

Cons: 
- Unable to represent data in editor since the serialization system does not support inheritance, `healAmount` and `damageAmount` are not visible and editable in `Ability`

- **Generic Effect type Ability**

``` c#

public class Ability<T> : ScriptableObject where T : Effect{
    public List<T> effects
}

public class DamageEffect : Effect{
    public int damageAmount;
    public override void Apply(){
        // Do damage
    }
}

public class HealEffect : Effect{
    public int healAmount;
    public override void Apply(){
        // Do heal
    }
}

public class DamageAbility : Ability<DamageEffect> {

}

public class HealAbility : Ability<HealEffect>{

}
```

Pros:
- Clean
- Individual Effect's data can be represented.

Cons:
- Need ability scriptable object class for each effect
- Difficult to support multiple effect ability

- **Effect prefab**

``` c#
public class Ability : ScriptableObject{
    //...
    public List<Effect> effects;

}

public class Effect : MonoBehaviour{
    public virtual void Apply(){}
}

public class DamageEffect : Effect{
    public int damageAmount;
    public override void Apply(){
        //Do damage
    }
}

```

Pros:
- Clean
- Individual Effect's data can be represented
- Single Ability scriptable class

Cons:
- Effect data not editable from ability scriptable object
- One prefab for each ability's effects


Currently I am using effect prefab since one effect prefab for one skill is pretty inevitable as the vfx and sfx are different for most of abilities. Using prefab I also have freedom to add custom behaviours in the prefab.

# Tasks
---

- [x] Client build and deploy flow
- [x] 2D Game basics (Control, character, camera)
- [x] Basic game loop
- [x] Continuous ability
- [ ] Rift rewards (Currencies, Equipments) 
    - [ ] Equipment and items
- [x] Character building (Level up, Shops)
    - [ ] Character Ability tree
    - [ ] Character HUD and menu for character building


- [x] DB setup
- [x] User register and authentication
- [x] Find a working http package for webgl
- [ ] Rift reward generator API
- [ ] ~~Character building API~~

Next step I will try to wrap up the character building and equipments.