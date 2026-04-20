# [Just use Postgres](https://mccue.dev/pages/8-16-24-just-use-postgres)

by: **Ethan McCue**

This is one part actionable advice, one part question for the audience.

Advice: When you are making a new application that requires persistent storage of data, like is the case for most web applications, your default choice should be `Postgres`.

### Why not `sqlite`?

`sqlite` is a pretty good database, but its data is stored in a single file.

This implies that whatever your application is, it is running on one machine and one machine only. Or at least one shared filesystem.

If you are making a desktop or mobile app, that's perfect. If you are making a website it might not be.

There are many success stories of using `sqlite` for a website, but they mostly involve people who set up their own servers and infrastructure. Platforms as a service-s like Heroku, Railway, Render, etc. generally expect you to use a database accessed over network boundary. It's not _wrong_ to give up some of the benefits of those platforms, but do consider if the benefits of `sqlite` are worth giving up platform provided automatic database backups and the ability to provision more than one application server.

[The official documentation](https://www.sqlite.org/whentouse.html) has a good guide with some more specifics.

### Why not `DynamoDB`, `Cassandra`, or `MongoDB`?

Wherever Ray Houlihan is, I hope he is having a good day.

I watch a lot of conference talks, but his [2018 DynamoDB Deep Dive](https://www.youtube.com/watch?v=HaEPXoXVf2k) might be the one I've watched the most. I know very few of you are going to watch an hour-long talk, but you really should. It's a good one.

The thrust of it is that databases that are in the same genre as `DynamoDB` - which includes `Cassandra` and `MongoDB` - are fantastic **if** - and this is a load bearing if:

- You know exactly what your app needs to do, up-front
- You know exactly what your access patterns will be, up-front
- You have a known need to scale to really large sizes of data
- You are okay giving up some level of consistency

This is because this sort of database is basically a giant distributed hash map. The only operations that work without needing to scan the entire database are lookups by partition key and scans that make use of a sort key.

Whatever queries you need to make, you need to encode that knowledge in one of those indexes before you store it. You want to store users and look them up by either first name or last name? Well you best have a sort key that looks like `<FIRST NAME>$<LAST NAME>`. Your access patterns should be baked into how you store your data. If your access patterns change significantly, you might need to reprocess all of your data.

It's annoying because, especially with `MongoDB`, people come into it having been sold on it being a more "flexible" database. Yes, you don't need to give it a schema. Yes, you can just dump untyped JSON into collections. No, this is not a flexible kind of database. It is an efficient one.

With a relational database you can go from getting all the pets of a person to getting all the owners of a pet by slapping an index or two on your tables. With this genre of NoSQL, that can be a tall order.

Its also not amazing if you need to run analytics queries. Arbitrary questions like "How many users signed up in the last month" can be trivially answered by writing a SQL query, perhaps on a read-replica if you are worried about running an expensive query on the same machine that is dealing with customer traffic. It's just outside the scope of this kind of database. You need to be ETL-ing your data out to handle it.

If you see a college student or fresh grad using `MongoDB` stop them. They need help. They have been led astray.

### Why not `Valkey`?

The artist formerly known as `Redis` is best known for being an efficient out-of-process cache. You compute something expensive once and slap it in `Valkey` so all 5 or so HTTP servers you have don't need to recompute it.

However, you _can_ use it as your primary database. It stores all its data in RAM, so it's pretty fast if you do that.

Obvious problems:

- You can only have so much RAM. You can have a lot more than you'd think, but its still pretty limited compared to hard drives.
- Same as the `DynamoDB`-likes, you need to make concessions on how you model your data.

### Why not `Datomic`?

If you already knew about this one, you get a gold star.

`Datomic` is a `NoSQL` database, but it is a relational one. The "up-front design" problems aren't there, and it does have some neat properties.

You don't store data in tables. It's all "entity-attribute-value-time" (EAVT) pairs. Instead of a person row with `id`, `name`, and `age` you store `1 :person/name "Beth"` and `1 :person/age 30`. Then your queries work off of "universal" indexes.

You don't need to coordinate with writers when making queries. You query the database "as-of" a given time. New data, even deletions (or as they call them "retractions"), don't actually delete old data.

But there are some significant problems

- It only works with JVM languages.
- Outside of `Clojure`, a relatively niche language, its API sucks.
- If you structure a query badly the error messages you get are terrible.
- The whole universe of tools that exist for SQL just aren't there.

### Why not `XTDB`?

`Clojure` people make a lot of databases.

`XTDB` is spiritually similar do `Datomic` but:

- There is an HTTP api, so you aren't locked to the JVM.
- It has two axes of time you can query against. "System Time" - when records were inserted - and "Valid Time."
- It has a SQL API.

The biggest points against it are:

- It's new. Its SQL API is something that popped up in the last year. It recently changed its whole storage model. Will the company behind it survive the next 10 years? Who knows!

Okay that's just one point. I'm sure I could think of more, but treat this as a stand-in for any recently developed database. The best predictor something will continue to exist into the future is how long it has existed. COBOL been around for decades, it will likely continue to exist for decades.

If you have persistent storage, you want as long a support term as you can get. You can certainly choose to pick a newer or experimental database for your app but, regardless of technical properties, that's a risky choice. It shouldn't be your default.

### Why not `Kafka`?

`Kafka` is an append only log. It can handle TBs of data. It is a very good append only log. It works amazingly well if you want to do event sourcing type stuff with data flowing in from multiple services maintained by multiple teams of humans.

But:

- Up to a certain scale, a table in Postgres works perfectly fine as an append only log.
- You likely do not have hundreds of people working on your product nor TBs of events flowing in.
- Making a Kafka consumer is a bit more error-prone than you'd expect. You need to keep track of your place in the log after all.
- Even when maintained by a cloud provider (and there are good managed `Kafka` services) its another piece of infrastructure you need to monitor.

### Why not `ElasticSearch`?

Is searching over data the primary function of your product?

If yes, `ElasticSearch` is going to give you some real pros. You will need to ETL your data into it and manage that whole process, but `ElasticSearch` is built for searching. It does searching good.

If no, `Postgres` will be fine. A sprinkling of `ilike` and the built-in [full text search](https://www.postgresql.org/docs/current/textsearch.html) is more than enough for most applications. You can always bolt on a dedicated search thing later.

### Why not `MSSQL` or `Oracle DB`?

Genuine question you should ask yourself: Are these worth the price tag?

I don't just mean the straight-up cost to license, but also the cost of lock-in. Once your data is in `Oracle DB` you are going to be paying Oracle forever. You are going to have to train your coders on its idiosyncrasies, forever. You are going to have to decide between enterprise features and your wallet, forever.

I know its super unlikely that you will contribute a patch to `Postgres`, so I won't pretend that there is some magic "power of open source" going on, but I think you should have a very specific need in mind to choose a proprietary DB. If you don't have some killer `MSSQL` feature that you simply cannot live without, don't use it.

### Why not `MySQL`?

This is the one that I need some audience help with.

`MySQL` is owned by Oracle. There are [features locked behind their enterprise editions](https://www.mysql.com/products/enterprise/compare/). To an extent you will have lock-in issues the same as any other DB.

But the free edition `MySQL` has also been used in an extremely wide range of things. It's been around for a long time. There are people who know how to work with it.

My problem is that I've only spent ~6 months of my professional career working with it. I genuinely don't know enough to compare it intelligently to `Postgres`.

I'm convinced it isn't secretly so much better that I am doing folks a disservice when telling them to use `Postgres`, and I do remember reading about how `Postgres` generally has better support for enforcing invariants in the DB itself, but I wouldn't mind being schooled a bit here.

> I can tell you why not MySQL (as someone who's had the dubious "pleasure" of using it as the main DB for the last 3+ years (due to application lock-in):
>
> - MySQL is a dated database with not-great design, which has been in barebones maintenance mode for years. It's not really improving -- Oracle has little to no incentive. MariaDB has advanced some ahead of it but is also well behind Postgres.
> - MySQL has a badly outdated query optimizer, meaning that with moderately complex queries you can get shockingly bad performance in some cases. I've seen queries go from seconds to minutes, and minutes to hours. Postgres on the other hand has a quite advanced query optimiser, going as far as using genetic algorithms to find the most efficient query plan.
>   - Lived experience in production: if PG gives bad performance it's because you didn't index something that needed it, or your query CAN'T be executed efficiently as written. MySQL will require you to provide query hints to avoid prod-crippling outages unless your queries stay rather simple.
> - MySQL administration, management, and monitoring tooling are very bare-bones and hacky. Postgres has quite advanced admin interfaces, but in MySQL things like querying the information schema, sys tables, or performance schema can come with quite painful performance hits or limited information
> - MySQL does not adhere to SQL standards as closely as Postgres does; it doesn't take much effort to find nonstandard behaviors -- or cases where MySQL implicitly does something
> - MySQL is less trustworthy in how it behaves; we've found quite a few undocumented or questionable behaviors, including cases where what are _supposed_ to be lockless schema changes totally lock up production. Table statistics updates in MySQL sometimes don't happen when they should.
> - Lots of trivial things in MySQL are more painful than they should be -- missing `IF EXISTS/IF NOT EXISTS` for idempotent schema changes (some objects support it, some do not), `CREATE OR REPLACE` for functions/sprocs, etc

### Why not some AI vector DB?

- Most are new. Remember the risks of using something new.
- AI is a bubble. A load-bearing bubble, but a bubble. Don't build a house on it if you can avoid it.
- Even if your business is another AI grift, you probably only need to `import openai`.

### Why not Google Sheets?

You're right. I can't think of any downsides. Go for it.
