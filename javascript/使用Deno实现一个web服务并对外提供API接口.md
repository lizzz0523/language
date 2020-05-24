> 最近Deno正式发布1.0版本，虽然嘴上说了学不动，但身体还是很诚实。在网上看到这篇比较基础的Deno教程，当中展示了如何使用Deno完成API接口的开发，既包含了web服务的创建，亦包含数据库的访问，十分值得分享，于是有了这个翻译版本。同时在学习的过程中，对文中的代码部分进行了一些布局上的优化，并上传到github中
> [https://github.com/lizzz0523/deno-example/tree/master/api-server](https://github.com/lizzz0523/deno-example/tree/master/api-server)
>
> [原文] [Deno web server + API Tutorial](https://crunchskills.com/deno-web-server-and-api-tutorial/)  
> [作者] Alex  
> [翻译] lizzzled


Deno使用rust实现，能提供一个类型安全的运行时，这使得它最近可谓一时无两。此外，只有在得到你明确的授权后，Deno才能访问你计算机上的资源，如果这听起来很对你的胃口，那么我想是时候抛弃node.js，投奔Deno的怀抱了。

### 安装 Deno

#### Shell
```bash
$ curl -fsSL https://deno.land/x/install/install.sh | sh
```

#### Powershell
```bash
$ iwr https://deno.land/x/install/install.ps1 -useb | iex
```

在你的系统上检查是否已经成功安装Deno
```bash
$ deno --help
```

如果出错，那很有可能是环境变量`PATH`设置问题，具体可以参考这里  
[https://deno.land/manual/getting_started/installation](https://deno.land/manual/getting_started/installation)

### 创建你的第一个Deno文件
```bash
$ touch api.ts
```

Deno引入外部依赖的方式与npm完全不同，下面的代码展示了如何在Deno中引入server模块
```typescript
import { serve } from 'https://deno.land/std@v0.42.0/http/server.ts';
```

以及引入sqlite模块用于实现数据库管理相关功能
```typescript
import { open, save } from "https://deno.land/x/sqlite/mod.ts";
```

### Hello World，让我们的服务器程序运行起来

在项目开始的时候，我们总是应该先写一些简单的代码来验证你的想法，就像下面的这段代码。你甚至可以考虑把这段代码上传到你的服务器上来进行测试
```typescript
const PORT = 8162;
const s = serve({ port: PORT });

console.log(`Listening on http://localhost:${PORT}`);

for await (const req of s) {
  req.respond({ body: 'Hello World\n' });
}
```

接下来，你可以尝试使用以下命令来运行这段代码
```bash
$ deno run --allow-net api.ts
```

其中的`--allow-net`标志，用于表示允许Deno访问网络，这是另一个Deno优于node的地方，Deno永远只能访问那些你明确授权的资源。  
现在尝试在浏览器中访问 http://localhost:8162，你应该可以看到页面能正常的打开。

### 开发API接口

接下来，让我们对请求对象进行进一步的处理
```typescript
for await (const req of s) {
  const url = req.url;
  req.respond({ body: `Hi there, accessing from ${url}` });
}
```

现在，你可以尝试在浏览器中访问任意路径，例如：  
http://localhost:8162/hello/world  

不难发现，`req.url`字段代表的就是我们正在访问的路径。

我们可以使用`?`对`req.url`字段进行分割，从而得到了url路径以及url参数两部分。这样我们就可以使用这些url参数，例如count参数，来指定某个api需用从数据库中获取多少条记录。
```typescript
const params = req.url.split('?');
```

别忘了要对分割结果进行简单的校验，以确认接收到的url是正常的
```typescript
if (params.length > 2) {
  req.respond(badRequest);
  continue;
}
```

上面的代码表示，当用户访问的url中包含多于一个`?`时，我们无法处理，并通过返回一个`badRequest`对象来告知用户。

由于我们在一个`for`循环中，所以这里可以简单的通过`continue`关键字来跳过之后的其他逻辑。

我们把`badRequest`对象的声明写在前面，这样后续的代码就可以直接使用它来对错误进行响应。
```typescript
 const badRequest = { body: 'Error 400 bad request.', status: 400 };
```

使用url参数部分来初始化Deno内建的`URLSearchParams`对象，这样能方便我们获取url参数的每一个字段
```typescript
const pathname = params[0];
const searchParams = new URLSearchParams(params[1]);
```

### 数据库

是时候加入数据库部分的逻辑了。在文件的顶部加入如下代码，用于打开数据库文件
```typescript
const db = await open('jobboard.db');
```

这同时意味这Deno要对本地文件进行读写操作。所以在运行Deno时，我们需要添加`--allow-read`以及`--allow-write`两个标志
```bash
$ deno run \
  --allow-net \
  --allow-read \
  --allow-write \
  api.ts
```

我们将会在我们的API接口中，返回招聘岗位的列表信息，当然也有相应的接口用于添加招聘岗位信息。

因此，我们首先需要加入以下代码，用于在必要时创建相关的数据库表
```typescript
db.query(
  `CREATE TABLE IF NOT EXISTS jobs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at DATETIME,
    last_updated DATETIME,
    active BOOLEAN,
    company_id INTEGER,
    apply_url TEXT,
    job_title CHARACTER(140),
    job_details TEXT,
    pay_range TEXT
  )`
);
db.query(
  `CREATE TABLE IF NOT EXISTS companies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    logo_url TEXT,
    name CHARACTER(50),
    description TEXT,
    created_at DATETIME,
    last_updated DATETIME,
    active BOOLEAN
  )`
);
```

### API接口

我们已经搭建好我们的数据库，接下来，我们需要两个接口，分别用于查看和添加招聘岗位信息。

在`for`循环中，加入简单的`if`语句来完成路由的逻辑
```typescript
if (pathname === '/api/v1/jobs') {
}
```

然后检查url参数中是否包含了`count`字段，如果是，则进一步的判断`count`字段的值是否大于`100`，我们并不希望用户一次能获取超过100条的记录。
```typescript
let count = 10; // base  
let requestCount = searchParams.get('count');
if (requestCount) {
  // enforcing max 100 record request
  if (parseInt(requestCount) > 100) {
    req.respond(badRequest);
    continue;
  } else {
    count = parseInt(requestCount);
  }
}
```

从数据库中，由于一个公司可能有多条招聘信息，但一条招聘信息只能属于一个公司，因此，我们把招聘信息和公司信息分别存储在`jobs`和`companies`两个表中，通过在`company`的`id`字段上对`jobs`表以及`companies`表进行`join`操作，我们就能直接获取到`job_title`字段以及`company`的`name`字段
```typescript
for (const [
  jobTitle,
  companyName,
] of db.query(
  `SELECT job_title, name
    FROM jobs JOIN companies ON company_id = companies.id
    ORDER BY jobs.id DESC
    LIMIT ?
  `,
  [count]
)) {
  results.push({
    company_name: companyName,
    job_title: jobTitle,
  });
}
```

拿到结果后，我们就能构造正确的响应
```typescript
req.respond({
  body: JSON.stringify(results),
  status: 200,
});
continue;
```

下面是完整代码展示
```typescript
if (pathname === '/api/v1/jobs') {
  let count = 10; // base  
  let requestCount = searchParams.get('count');
  if (requestCount) {
    // enforcing max 100 record request
    if (parseInt(requestCount) > 100) {
      req.respond(badRequest);
      continue;
    } else {
      count = parseInt(requestCount);
    }
  }
  const results = [];
  for (const [
    jobTitle,
    companyName,
  ] of db.query(
    `SELECT job_title, name
      FROM jobs JOIN companies ON company_id = companies.id
      ORDER BY jobs.id DESC
      LIMIT ?
    `,
    [count]
  )) {
    results.push({
      company_name: companyName,
      job_title: jobTitle,
    });
  }
  req.respond({
    body: JSON.stringify(results),
    status: 200,
  });
  continue;
}
```

现在，我们要开始实现另一个接口了，这个接口用于向数据库中添加招聘信息记录，并且出于安全起见，加入了密码保护
```typescript
if (url == '/api/v1/jobs/add') {
}
```

这里我们暂时把密码硬编码到源文件中，后续可以进一步的优化。如果用户在url参数中包含了`pw`字段，我们就可以检查用户提供的密码是否正确
```typescript
const password = 'supersecurepassword'; // set our pasword
let passwordValid = false; // initialize to false
let requestPassword = searchParams.get('pw'); // check the search param pw if its a valid password
if (requestPassword) {
  if (requestPassword === password) {
    passwordValid = true;
  } 
}
```

如果用户提供的密码不正确，我们直接返回`'Not Allowed'`，并且结束处理
```typescript
if (passwordValid == false) {
  req.respond({
    body: 'Not Allowed',
    status: 405,
  });
  continue;
}
```

如果用户提供的密码正确，我们就可以把url参数中包含的其他参数安全的添加到数据库中。  
注意，这里我们需要使用`name`字段在`companies`表中反查出`id`字段，并且作为`jobs`表的`company_id`字段写入数据库
```typescript
const applyUrl = searchParams.get('apply_url');
const jobTitle = searchParams.get('job_title');
const company = searchParams.get('company');
const details = searchParams.get('details');
const payRange = searchParams.get('pay_range');
let companyId;
// To get the company ID we need to know if from the other table
for (const [id] of db.query(
  `SELECT id
    FROM companies
    WHERE name = ?
  `,
  [company]
)) {
  companyId = id;
}
// companies are added in a different endpoint
if (!companyId) {
  req.respond({
    body: 'Company Name not specified.',
    status: 400,
  });
  continue;
}
// write into database
db.query(
  `INSERT INTO jobs (
    company_id,
    apply_url,
    job_title,
    details,
    pay_range,
    created_at,
    last_updated,
    active
  ) VALUES (
    ?,?,?,?,?,?,?,?
  )`,
  [
    companyId,
    applyUrl,
    jobTitle,
    details,
    payRange,
    time,
    time,
    1
  ]
);
req.respond({
  status: 200,
});
```

下面是完整代码展示
```typescript
if (url == '/api/v1/jobs/add') {
  const password = 'supersecurepassword'; // set our pasword
  let passwordValid = false; // initialize to false
  let requestPassword = searchParams.get('pw'); // check the search param pw if its a valid password
  if (requestPassword) {
    if (requestPassword === password) {
      passwordValid = true;
    } 
  }
  if (passwordValid == false) {
    req.respond({
      body: 'Not Allowed',
      status: 405,
    });
    continue;
  }
  const applyUrl = searchParams.get('apply_url');
  const jobTitle = searchParams.get('job_title');
  const company = searchParams.get('company');
  const details = searchParams.get('details');
  const payRange = searchParams.get('pay_range');
  let companyId;
  // To get the company ID we need to know if from the other table
  for (const [id] of db.query(
    `SELECT id
      FROM companies
      WHERE name = ?
    `,
    [company]
  )) {
    companyId = id;
  }
  // companies are added in a different endpoint
  if (!companyId) {
    req.respond({
      body: 'Company Name not specified.',
      status: 400,
    });
    continue;
  }
  // write into database
  db.query(
    `INSERT INTO jobs (
      company_id,
      apply_url,
      job_title,
      details,
      pay_range,
      created_at,
      last_updated,
      active
    ) VALUES (
      ?,?,?,?,?,?,?,?
    )`,
    [
      companyId,
      applyUrl,
      jobTitle,
      details,
      payRange,
      time,
      time,
      1
    ]
  );
  req.respond({
    status: 200,
  });
}
```

### API接口完成

在文件的最后，我们只需简单的执行关闭数据库操作即可
```typescript
await save(db);
db.close();
```

上面的例子中，我们展示如何创建普通接口和有密码保护的接口，同时也展示了如何从url参数中获取任意字段，接下来你就可以基于这些能力创建任意你想要的API接口。尽情释放你的想象力。