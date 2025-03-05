---
layout: single
title: Arkademy Devlog 1
date: 2025-03-05 11:34 +0800
categories: Personal
tags: ["Devlog","Arkademy"]
toc: true
toc_lable: "Contents"
---

# Updates
---
- Backend setup
- CI setup for Backend
- Backend Service in Game

# Backend
---
As mentioned in the previous dev log, the backend is currently implemented using Golang with Mongodb. The coding part is fairly easy since I asked Copilot for the structure and some utility code like hashing a password or generate token in golang.

``` go

import (
	"crypto/sha256"
	"encoding/hex"
	"time"

	"github.com/dgrijalva/jwt-go"
)

func hashPassword(password string) string {
	hash := sha256.Sum256([]byte(password))
	return hex.EncodeToString(hash[:])
}

func generateToken(username string) (string, error) {
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"username": username,
		"exp":      time.Now().Add(time.Hour * 72).Unix(),
	})
	return token.SignedString(jwtKey)
}

```

## Model
Currently the model design is fairly simple since most of the game play logics are only running on the client side, the server currently only serves authentication and data storage purpose. (More like a cloud save)

- **User** is the main document in the database collection. It stores username and password for basic authentication, as well as play record which records the user`s game play progress.

- **PlayerRecord** is just a struct holds a list of CharacterRecords.

- **CharacterRecord** is the struct holds a single character`s data stored unstructured in database as a plain JSON since the structure might change very frequently, I dont want to update server code every time I update the game logic.

## Api
For register and login api it is quite straight forward, just two POST requests without additional checks. ~~I should though~~ 

For user related APIs, paramter is not used to indicate which user is the target for api. Instead I just simply checking the token and find the user indicate in the token and treat it as the target. The reason is for now any user related operation only require their own contens, although this needs to be changed in the future when the user can check other user`s info or when I need an admin system.

A simple token auth middleware is employed for the user related APIs.

``` go
func AuthMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		tokenString := c.GetHeader("Authorization")
		if tokenString == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "Authorization header is required"})
			c.Abort()
			return
		}

		token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, errors.New("Invalid signing method")
			}
			return jwtKey, nil
		})

		if err != nil || !token.Valid {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
			c.Abort()
			return
		}

		claims, ok := token.Claims.(jwt.MapClaims)
		if !ok || !token.Valid {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
			c.Abort()
			return
		}
		username := claims["username"].(string)
		var user User
		err = userCollection.FindOne(context.TODO(), bson.M{"username": username}).Decode(&user)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": "User not found"})
			return
		}
		c.Set("user", user)
		c.Next()
	}
}
```

### Token Not required

`POST` `/` `register` 

```json
{
    "username": "your_name",
    "password": "your_password"
}
```

`POST` `/` `login`

```json
{
    "username": "your_name",
    "password": "your_password"
}
```

### Token required

`GET` `/` `user`

`PATCH` `/` `player`

``` json
{
    "Characters": []
}
```

## CI/CD

Currently the server is using self hosted runner to build with github action, build on every main branch update.

## Hosting

Currently the server is hosted in my home Mac mini. Initially I tried to run it directly, but it does not work well with github action. After every CI build, I tried to launch the server executable, but it will be killed immediately by the CI clean up stage. 

In order to restart the server automatically after each CI build, I have to use Docker, so that I can just docker run in the end. This creates a few problems but solved by using specific Docker build and run setup.

- To remove old Docker image after build
``` yaml
run: docker image prune -f
```

- To stop current running container
```yaml
run: docker stop container-api-arkademy || true
```

My docker container name is *container-api-arkademy*, ```|| true``` is appended to avoid error produced when there is no running container with this name.

- To run container
```yaml
run: docker run --rm -d -p 8080:8080 \
        -e MONGO_URI_LOCAL=${MONGO_URI_LOCAL} \
        -e JWT_SECRET=${JWT_SECRET} \
        --name container-api-arkademy image-api-arkademy
```
- `--rm` to remove the container when it failed or stopped
- `-d` to run in background and detached

For MongoDB it is currently running directly in my Mac mini, since the setup for db is quite simple, it is not part of the CI yet, just running persistently outside of container. This creates problem in the beginning as I cant get the server in container to connect to it. 

Later I find the solution to be just use `host.docker.internal` in the connection url for mongodb like `mongodb://host.docker.internal:27021` in the container. 

Since I use .env file to setup the mongodb connection in server code, I setup the CI to pass the url to docker and Dockerfile to generate the .env file for the server to use. At the same time, I can different .env file when I develop on different system. To connect to mongodb from different machine in same network, the mongo config need to use

```yaml
net:
    bindIp: 0.0.0.0
```

# Client
---

Apparently I cant use `System.Net` in Webgl, I have to find some other way to do it...



# Tasks
---

- [x] Client build and deploy flow
- [x] 2D Game basics (Control, character, camera)
- [x] Basic game loop
- [ ] Rift rewards (Currencies, Equipments) 
- [ ] Character building (Level up, Shops)

- [x] DB setup
- [x] User register and authentication
- [ ] **Find a working http package for webgl** 
- [ ] Rift reward generator API
- [ ] Character building API


Next I should be spending a lot of time to implement the character building and rewards system since they depends alot on game play contents like abilities and equipments.