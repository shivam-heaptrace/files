Agent Plugins for AWS :

### **1. What problem are Agent Plugins for AWS designed to solve?**

Agent Plugins for AWS are designed to simplify and speed up cloud deployment workflows by extending AI coding agents with specialized AWS skills so they can handle researching service options, estimating costs, and writing infrastructure-as-code (IAC)—tasks that typically slow down development workflows. This reduces the need for manually pasting long guidance into prompts and helps standardize agent behavior across teams. ([Amazon Web Services, Inc.][1])

---

### **2. What components can an agent plugin include? Explain each one.**

According to the article, an agent plugin can include: ([Amazon Web Services, Inc.][1])

1. **Agent skills** – Structured workflows and best-practice playbooks that guide an AI through complex tasks like deployment, code review, or architecture planning. They encode domain expertise as step-by-step processes. ([Amazon Web Services, Inc.][1])

2. **MCP servers** – Connections to external services, data sources, and APIs. They provide the assistant with access to live documentation, pricing data, and other resources at runtime. ([Amazon Web Services, Inc.][1])

3. **Hooks** – Automation and guardrails that run on developer actions. Hooks can validate changes, enforce standards, or trigger workflows automatically. ([Amazon Web Services, Inc.][1])

4. **References** – Documentation, configuration defaults, and knowledge that the agent skill can consult. These make the agent skills “smarter” without needing extra prompt context. ([Amazon Web Services, Inc.][1])

---

### **3. What is the purpose of the deploy-on-aws plugin?**

The deploy-on-aws plugin gives AI coding agents the ability to deploy applications to AWS by providing architecture recommendations, cost estimates, and infrastructure-as-code generation. ([Amazon Web Services, Inc.][1])

---

### **4. What are the five phases the deploy-on-aws plugin performs?**

The deploy-on-aws plugin performs these five phases: ([Amazon Web Services, Inc.][1])

1. **Analyze** – Scans the codebase for framework, database, and dependencies.
2. **Recommend** – Selects optimal AWS services with concise rationale.
3. **Estimate** – Shows projected monthly costs before committing.
4. **Generate** – Writes AWS CDK or CloudFormation infrastructure code.
5. **Deploy** – Executes the deployment after user confirmation. ([Amazon Web Services, Inc.][1])

---

### **5. Which AI coding tools are mentioned as compatible in the article?**

The article mentions that Agent Plugins for AWS are currently supported in **Claude Code** and **Cursor**. ([Amazon Web Services, Inc.][1])

---

### **6. Describe the example application used in the blog post and how it was deployed.**

The example application is a full-stack project consisting of:

- An **Express.js REST API**
- A **PostgreSQL database**
- A **React frontend**

The developer used Cursor or Claude Code with the deploy-on-aws plugin installed and entered a natural language request (_“Deploy this Express app to AWS”_). The plugin then analyzed the codebase, recommended AWS services, estimated costs, generated IAC code (including AWS CDK, Dockerfile, database migrations, environment config, and CI/CD workflows), and deployed the infrastructure. Within minutes, the app was live with backend and frontend URLs, database details, monitoring dashboards, and cost tracking set up. ([Amazon Web Services, Inc.][1])

---

### **7. Which AWS services were recommended in the example deployment?**

The agent recommended:

- **AWS App Runner** for the Express backend
- **Amazon RDS PostgreSQL** for the database
- **Amazon CloudFront + S3** for the React frontend
- **AWS Secrets Manager** for database credentials. ([Amazon Web Services, Inc.][1])

---

### **8. How does the plugin estimate costs according to the article?**

The plugin uses real-time pricing data from the **AWS Pricing MCP server** to provide a projected monthly cost estimate before committing to infrastructure. ([Amazon Web Services, Inc.][1])

---

### **9. What role do MCP servers play in agent plugins?**

MCP servers connect the agent plugin to external services, data sources, and APIs, giving the assistant access to live documentation, pricing data, and other runtime resources. ([Amazon Web Services, Inc.][1])

---

### **10. What are hooks and how are they used in the plugin system?**

Hooks are automation and guardrails that run on developer actions. They can be used to validate changes, enforce standards, or trigger workflows automatically. ([Amazon Web Services, Inc.][1])

---

### **11. What infrastructure-as-code tools are mentioned in the article?**

The article mentions **AWS CDK (Cloud Development Kit)** and **AWS CloudFormation** as the two infrastructure-as-code approaches used by the deploy-on-aws plugin. ([Amazon Web Services, Inc.][1])

---

### **12. Does the article mention Terraform support?**

It **does not** mention Terraform. The only IAC tools referenced are AWS CDK and AWS CloudFormation. ([Amazon Web Services, Inc.][1])

---

### **13. Does the article describe support for Azure or Google Cloud?**

No, the article does **not** mention support for Azure or Google Cloud. ([Amazon Web Services, Inc.][1])

---

### **14. What security best practices does the article recommend?**

The article recommends:

- Always review generated code before deployment against your own constraints for security, cost, and resilience.
- Use plugins as accelerators, not replacements for developer judgment.
- Keep plugins updated with the latest AWS best practices.
- Follow the principle of least privilege when configuring AWS credentials.
- Run security scanning tools on generated infrastructure code. ([Amazon Web Services, Inc.][1])

---
