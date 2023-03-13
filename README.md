# hxp-CTF-2022-Review

## Introduction

This is hxp CTF 2022 Review Web Category: valentine

## Description

Create an awesome template for your valentine and share it with the world! 

Seems like we have to perform Server Side Template Injection (SSTI)

## Phase I

![image](https://user-images.githubusercontent.com/79100627/224588481-abbcbfe6-e55d-4aad-a58f-5285d33b85ec.png)

![image](https://user-images.githubusercontent.com/79100627/224588835-fe87180a-23b3-43f5-93b3-a4c6a0edba05.png)

When I type any input, valentine card with the name will show up:

![image](https://user-images.githubusercontent.com/79100627/224588907-22689c75-cfef-48be-bfd1-c9dd9017b1f1.png)

When I delete the inside html code with the <%= process %>, the validation will happen.

![image](https://user-images.githubusercontent.com/79100627/224589030-9e202c87-6a27-4d62-ac77-f0d7093a9fde.png)

## Validation happens

![image](https://user-images.githubusercontent.com/79100627/224589096-9ad5d148-9372-4794-aab5-c54ee8aa9555.png)

## Phase II Code Review

### Docker File
```
# see docker-compose.yml

FROM node:current-bullseye
ENV NODE_ENV=production
WORKDIR /app

COPY package.json package-lock.json app.js index.html /app/
COPY flag.txt readflag /
RUN npm install

RUN mkdir views
RUN chown node:node views

RUN chown root:root /flag.txt && chmod 400 /flag.txt
RUN chown root:root /readflag && chmod 4555 /readflag

EXPOSE 3000

USER node
CMD node app.js
```
I guess we can access to the flag with the /readflag path.

### App.js
```
var express = require('express');
var bodyParser = require('body-parser')
const crypto = require("crypto");
var path = require('path');
const fs = require('fs');

var app = express();
viewsFolder = path.join(__dirname, 'views');

if (!fs.existsSync(viewsFolder)) {
  fs.mkdirSync(viewsFolder);
}

app.set('views', viewsFolder);
app.set('view engine', 'ejs');

app.use(bodyParser.urlencoded({ extended: false }))

app.post('/template', function(req, res) {
  let tmpl = req.body.tmpl;
  let i = -1;
  while((i = tmpl.indexOf("<%", i+1)) >= 0) {
    if (tmpl.substring(i, i+11) !== "<%= name %>") {
      res.status(400).send({message:"Only '<%= name %>' is allowed."});
      return;
    }
  }
  let uuid;
  do {
    uuid = crypto.randomUUID();
  } while (fs.existsSync(`views/${uuid}.ejs`))

  try {
    fs.writeFileSync(`views/${uuid}.ejs`, tmpl);
  } catch(err) {
    res.status(500).send("Failed to write Valentine's card");
    return;
  }
  let name = req.body.name ?? '';
  return res.redirect(`/${uuid}?name=${name}`);
});

app.get('/:template', function(req, res) {
  let query = req.query;
  let template = req.params.template
  if (!/^[0-9A-F]{8}-[0-9A-F]{4}-[4][0-9A-F]{3}-[89AB][0-9A-F]{3}-[0-9A-F]{12}$/i.test(template)) {
    res.status(400).send("Not a valid card id")
    return;
  }
  if (!fs.existsSync(`views/${template}.ejs`)) {
    res.status(400).send('Valentine\'s card does not exist')
    return;
  }
  if (!query['name']) {
    query['name'] = ''
  }
  return res.render(template, query);
});

app.get('/', function(req, res) {
  return res.sendFile('./index.html', {root: __dirname});
});

app.listen(process.env.PORT || 3000);
```

If we look at the App.js, fs is the file system module in node.js and this app is using 'ejs' module. <br/> 
'ejs' module can be find here: https://ejs.co/

There are three paths in this app.js 
- /template
- /:template
- / (main) 

### /template is create template 
![image](https://user-images.githubusercontent.com/79100627/224590609-80514dc7-329c-430c-90a5-b9b609383b83.png)

This /template filters any symbol except for <%= name %>. Therefore we are only allowed to use a name. <br/>
After the template is created, it will created in views/${uuid}.ejs then it will redirect /${uuid}?name=${name} -> /:template


### /:template is render template 
![image](https://user-images.githubusercontent.com/79100627/224590646-b0520e7a-0f67-4722-b6da-8ab1bc9d98ba.png)

in this path it make sure its a valid make uuid. if file exist. If file exist its going to take the query name, if it does not exist it will set equal "" <br/>

## Phase III bypass <%= name %>

![image](https://user-images.githubusercontent.com/79100627/224591464-44c316d7-e8a8-4985-9d38-7aa269585e70.png)

if we can bypass the <%= name %> we can process SSTI injection -> <%= process %> <br/>
in this code, we can take any parameter in query then it will send res.render(template, query)

![image](https://user-images.githubusercontent.com/79100627/224591715-f59bfa70-fef8-4214-b51b-8addd6a2e1be.png)

If we look at the feature, there is a  Custom delimiters (e.g., use [? ?] instead of <% %>)

![image](https://user-images.githubusercontent.com/79100627/224591850-4d977123-7363-4a72-922d-865662612a78.png)

we have to add the option. In the return res.render(template, { delimiter: "$" }); -> we can bypass the delimiter 

![image](https://user-images.githubusercontent.com/79100627/224592233-e4fca135-44ea-4b85-80b2-36afe5dfc692.png)

This is the payload for the SSTI injection 


