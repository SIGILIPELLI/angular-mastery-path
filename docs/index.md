---
title: "Free Angular Course: Signals, Signal Forms, SSR, Testing & Deployment"
description: "Free Angular 22 course: 36 lessons, 3 projects and 1 capstone on components, signals, signal forms, resources, routing, SSR, testing, i18n and production deployment."
---

# Angular Mastery Path

Angular is the framework you reach for when an application has to be big, long-lived and
maintained by more than one team. It gives you the whole toolkit in one coordinated
release — components, a reactivity model, dependency injection, a router, forms, an HTTP
client, server-side rendering, testing and a CLI that can upgrade your code for you. This
course teaches **modern Angular**: standalone components, **signals** everywhere, zoneless
change detection, the new template control flow, **signal forms**, **resources** for async
data, and per-route server rendering — and, just as importantly, the machinery underneath,
so you can reason about what your app is doing instead of copying patterns.

The course targets **Angular 22.2** (the version on npm while it was written, with
TypeScript 6.0 and Vitest 5). Level 1 builds a routed reading-list app from scratch: the
CLI, components and templates, signals, control flow, inputs and outputs, services, routing
and `HttpClient`. Level 2 makes apps real: reactive and signal forms, dependency injection in
depth, RxJS and interop, `resource`/`httpResource`, guards and resolvers, interceptors, pipes
and directives — ending with an issue tracker. Level 3 goes under the hood: change detection
without Zone.js, composition, state management, `@defer`, performance, SSR and hydration,
testing, accessibility and security — ending with a server-rendered catalog. Level 4 is about
production: build configurations, libraries, i18n, custom elements, upgrades, architecture,
deployment, error handling and a readiness review, finishing with a capstone.

Every lesson has a **How It Actually Works** section — what the compiler turns a template
into, why a plain field changed in `setTimeout` doesn't update the screen any more, how a
signal knows who depends on it, why `track $index` moves your typed text to the wrong row,
how hydration adopts server HTML instead of rebuilding it.

!!! note "Verified against real builds — and honest about what moves"
    The code in this course was compiled with `ng build`, tested with `ng test` and, for
    anything visual, driven in a real browser while writing. Terminal and browser output
    shown on these pages is output we actually observed — including the places where our
    first attempt was wrong, because those are the lessons worth learning. Angular
    changes defaults between major versions (new projects became zoneless, `OnPush`
    became the default, `@Service()` arrived in 22), so lessons say which behaviour
    belongs to which version. Where something couldn't be run locally (a Docker image,
    hosting-provider specifics), the lesson says so.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | What Angular is, the CLI, components & templates, signals, control flow, inputs/outputs/`model()`, services & DI, routing, `HttpClient` | 9 topics + 1 project (a reading-list app with live search) |
| [Level 2 · Intermediate](level-2/index.md) | Reactive forms, signal forms, DI in depth, RxJS, signals ↔ RxJS interop, `resource` & `httpResource`, guards & resolvers, interceptors, pipes & directives | 9 topics + 1 project (an issue tracker with an in-memory API) |
| [Level 3 · Advanced](level-3/index.md) | Change detection & zoneless, projection & queries, state management, `@defer`, performance, SSR & hydration, testing, accessibility, security | 9 topics + 1 project (a server-rendered product catalog) |
| [Level 4 · Master](level-4/index.md) | Build configurations, libraries, i18n, Angular Elements, upgrades & migrations, architecture, deployment, observability, production readiness | 9 topics + 1 capstone (the catalog taken to production standard) |

## How to use this site

- **Build as you read.** Create a project in Level 1, lesson 02 and keep it open. Code
  blocks are titled with their file path (`src/app/book-store.ts`) so you know exactly
  where each piece lives.
- **Run `ng build`, not just `ng serve`.** The production build is where you see bundle
  sizes, lazy chunks and budget errors, and many lessons ask you to read its output.
- **Know TypeScript first.** Angular is TypeScript everywhere; if classes, interfaces,
  union types and generics are new, start with
  [TypeScript Mastery Path](https://learnmasterypath.com/typescript-mastery-path/). For
  browser fundamentals see
  [JavaScript Mastery Path](https://learnmasterypath.com/javascript-mastery-path/); for
  the server runtime behind SSR, [Node.js Mastery Path](https://learnmasterypath.com/nodejs-mastery-path/);
  for end-to-end testing, [Playwright Mastery Path](https://learnmasterypath.com/playwright-mastery-path/);
  and to compare frameworks, [React Mastery Path](https://learnmasterypath.com/react-mastery-path/)
  builds similar projects.
- Do the **Exercise** at the end of each lesson; the module-10 projects build on them.

Start here → [Level 1 · Entry](level-1/index.md)


🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@learnmasterypath) on YouTube — Shorts and full walkthroughs of lessons across the series. [▶ Subscribe free](https://www.youtube.com/@learnmasterypath?sub_confirmation=1)

## More from the Mastery Path series

Free, structured, module-wise training across 101 other languages, platforms and disciplines — expand a category to browse:

<div class="mastery-tracks">
<details class="mastery-card">
<summary>Languages <span class="mastery-count">20</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/python-mastery-path/">🐍 Python</a>
  <a href="https://learnmasterypath.com/java-mastery-path/">☕ Java</a>
  <a href="https://learnmasterypath.com/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://learnmasterypath.com/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://learnmasterypath.com/csharp-mastery-path/">🔵 C#</a>
  <a href="https://learnmasterypath.com/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://learnmasterypath.com/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://learnmasterypath.com/c-mastery-path/">🇨 C</a>
  <a href="https://learnmasterypath.com/cpp-mastery-path/">➕ C++</a>
  <a href="https://learnmasterypath.com/go-mastery-path/">🐹 Go</a>
  <a href="https://learnmasterypath.com/rust-mastery-path/">🦀 Rust</a>
  <a href="https://learnmasterypath.com/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://learnmasterypath.com/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://learnmasterypath.com/php-mastery-path/">🐘 PHP</a>
  <a href="https://learnmasterypath.com/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://learnmasterypath.com/swift-mastery-path/">🐦 Swift</a>
  <a href="https://learnmasterypath.com/dart-mastery-path/">🎯 Dart</a>
  <a href="https://learnmasterypath.com/scala-mastery-path/">🔴 Scala</a>
  <a href="https://learnmasterypath.com/r-mastery-path/">📊 R</a>
  <a href="https://learnmasterypath.com/matlab-mastery-path/">🟧 MATLAB</a>
</div>
</details>
<details class="mastery-card">
<summary>Web Frameworks <span class="mastery-count">10</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/react-mastery-path/">⚛️ React</a>
  <a href="https://learnmasterypath.com/nextjs-mastery-path/">▲ Next.js</a>
  <a href="https://learnmasterypath.com/vue-mastery-path/">💚 Vue.js</a>
  <a href="https://learnmasterypath.com/html-css-mastery-path/">🎨 HTML & CSS</a>
  <a href="https://learnmasterypath.com/tailwind-mastery-path/">🌬️ Tailwind CSS</a>
  <a href="https://learnmasterypath.com/django-mastery-path/">🎸 Django</a>
  <a href="https://learnmasterypath.com/fastapi-mastery-path/">🚀 FastAPI</a>
  <a href="https://learnmasterypath.com/graphql-mastery-path/">🕸️ GraphQL</a>
  <a href="https://learnmasterypath.com/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://learnmasterypath.com/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
</details>
<details class="mastery-card">
<summary>Mobile <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/flutter-mastery-path/">💙 Flutter</a>
  <a href="https://learnmasterypath.com/react-native-mastery-path/">📲 React Native</a>
</div>
</details>
<details class="mastery-card">
<summary>Testing & QA <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://learnmasterypath.com/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://learnmasterypath.com/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://learnmasterypath.com/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
  <a href="https://learnmasterypath.com/selenium-mastery-path/">🧭 Selenium</a>
  <a href="https://learnmasterypath.com/cypress-mastery-path/">🌲 Cypress</a>
</div>
</details>
<details class="mastery-card">
<summary>Security <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
  <a href="https://learnmasterypath.com/ethical-hacking-mastery-path/">🎯 Ethical Hacking</a>
</div>
</details>
<details class="mastery-card">
<summary>Cloud Platforms <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/aws-mastery-path/">☁️ AWS</a>
  <a href="https://learnmasterypath.com/azure-mastery-path/">☁️ Azure</a>
  <a href="https://learnmasterypath.com/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://learnmasterypath.com/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://learnmasterypath.com/adobe-mastery-path/">🎨 Adobe</a>
</div>
</details>
<details class="mastery-card">
<summary>Data & Analytics <span class="mastery-count">7</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://learnmasterypath.com/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://learnmasterypath.com/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://learnmasterypath.com/excel-mastery-path/">📗 Excel</a>
  <a href="https://learnmasterypath.com/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://learnmasterypath.com/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
  <a href="https://learnmasterypath.com/power-bi-mastery-path/">📉 Power BI</a>
</div>
</details>
<details class="mastery-card">
<summary>Databases <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/postgresql-mastery-path/">🐘 PostgreSQL</a>
  <a href="https://learnmasterypath.com/mongodb-mastery-path/">🍃 MongoDB</a>
  <a href="https://learnmasterypath.com/redis-mastery-path/">🟥 Redis</a>
  <a href="https://learnmasterypath.com/snowflake-mastery-path/">❄️ Snowflake</a>
  <a href="https://learnmasterypath.com/databricks-mastery-path/">🧱 Databricks</a>
  <a href="https://learnmasterypath.com/python-mastery-path/vector-databases-mastery-path/">🧭 Vector Databases</a>
</div>
</details>
<details class="mastery-card">
<summary>AI / ML / LLM <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://learnmasterypath.com/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://learnmasterypath.com/rag-mastery-path/">📚 RAG</a>
  <a href="https://learnmasterypath.com/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://learnmasterypath.com/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://learnmasterypath.com/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://learnmasterypath.com/ml-math-mastery-path/">➗ ML Math Foundations</a>
  <a href="https://learnmasterypath.com/prompt-engineering-mastery-path/">✍️ Prompt Engineering</a>
  <a href="https://learnmasterypath.com/ai-agents-mastery-path/">🕹️ AI Agents</a>
  <a href="https://learnmasterypath.com/mlops-mastery-path/">🔁 MLOps</a>
  <a href="https://learnmasterypath.com/computer-vision-mastery-path/">👁️ Computer Vision</a>
  <a href="https://learnmasterypath.com/python-mastery-path/mcp-mastery-path/">🔌 MCP</a>
</div>
</details>
<details class="mastery-card">
<summary>Embedded Systems <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://learnmasterypath.com/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://learnmasterypath.com/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://learnmasterypath.com/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://learnmasterypath.com/s32k-mastery-path/">🔧 S32K</a>
</div>
</details>
<details class="mastery-card">
<summary>Leadership & Management <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://learnmasterypath.com/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://learnmasterypath.com/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://learnmasterypath.com/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://learnmasterypath.com/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
</details>
<details class="mastery-card">
<summary>Professional Skills <span class="mastery-count">4</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://learnmasterypath.com/workday-mastery-path/">🧑‍💼 Workday</a>
  <a href="https://learnmasterypath.com/digital-marketing-mastery-path/">📣 Digital Marketing & SEO</a>
  <a href="https://learnmasterypath.com/tally-gst-mastery-path/">🧾 Tally Prime & GST</a>
</div>
</details>
<details class="mastery-card">
<summary>Careers & Interviews <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://learnmasterypath.com/system-design-mastery-path/">🏗️ System Design</a>
</div>
</details>
<details class="mastery-card">
<summary>Process & APIs <span class="mastery-count">3</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://learnmasterypath.com/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://learnmasterypath.com/playwright-mastery-path/">🎭 Playwright</a>
</div>
</details>
<details class="mastery-card">
<summary>Infrastructure & Ops <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://learnmasterypath.com/linux-mastery-path/">🐧 Linux</a>
  <a href="https://learnmasterypath.com/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://learnmasterypath.com/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://learnmasterypath.com/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://learnmasterypath.com/docker-mastery-path/">🐳 Docker</a>
  <a href="https://learnmasterypath.com/github-mastery-path/">🐙 GitHub & Git</a>
  <a href="https://learnmasterypath.com/cicd-mastery-path/">🔁 CI/CD</a>
  <a href="https://learnmasterypath.com/ansible-mastery-path/">🤖 Ansible</a>
  <a href="https://learnmasterypath.com/kafka-mastery-path/">📨 Kafka</a>
  <a href="https://learnmasterypath.com/microservices-mastery-path/">🧩 Microservices</a>
  <a href="https://learnmasterypath.com/observability-mastery-path/">📈 Observability</a>
</div>
</details>
</div>
