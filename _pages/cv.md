---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* B.S. in Computer Engineering, Unisinos, 2024

Work experience
======
* Senior Database Reliability Engineer, Fireflies.ai
  * Abr 2025 - Present
  * Location: San Francisco, CA

* Founder, Goat IT
  * Aug 2023 - Present
  * Location: Campo Bom, Brazil
  
    As the Founder of Goat IT, I lead a team of experts in providing consultancy services to companies seeking to optimize, automate, and scale their IT operations. Our consultancy services are tailored to each client's unique needs, focusing on delivering solutions that enhance performance, drive innovation, and ensure the reliability and scalability of their IT systems.

    We specialize in database technologies, Infrastructure as Code (IaC), cloud platforms, artificial intelligence, software engineering, and various other technologies. Our team of professionals brings a wealth of experience and expertise to each project, working closely with clients to understand their requirements and develop customized solutions that meet their specific goals and objectives.

    Our consultancy services cover a wide range of areas, including database optimization, cloud migration, automation, security, performance tuning, and more. We work with clients across various industries, providing strategic guidance, technical expertise, and hands-on support to help them achieve their IT objectives and drive business growth.

    By leveraging our deep industry knowledge, technical skills, and innovative approach, we help our clients stay ahead of the curve and adapt to the rapidly evolving technology landscape. Our focus on collaboration, innovation, and excellence sets us apart and enables us to deliver exceptional results for our clients.

* Staff Database Reliability Engineer, PicPay
  * Sep 2023 - Mar 2025
  * Location: São Paulo, Brazil
  
    As a Staff Database Reliability Engineer (DBRE) at PicPay, I work with my team to create guardrails and CI/CD pipelines for databases, using tools like Terraform, Atlantis Pipeline, Tekton, GitHub Actions, and Backstage. Our goal is to deliver cloud infrastructure as a platform, incorporating guardrails, best practices, standard configurations, and pre-set user permissions. This enables developers to create and use the platform directly through Backstage.

    Additionally, we create database versioning pipelines using Flyway, integrated with ArgoCD for applications. We develop and maintain various applications and automations running on Lambda Functions or Kubernetes (EKS), using Helm Charts and ArgoCD for management. These applications reduce costs, integrate systems, generate information and alerts, improve operational efficiency, and support informed decision-making.

    I utilize Golang, Python, and JavaScript to develop some of these automations and integrations, enhancing the functionality and interoperability of our systems.

    We consult for teams facing challenges with services like DynamoDB, Redis, MongoDB, Aurora/RDS MySQL and PostgreSQL, Oracle Exadata, Oracle GoldenGate, and Keyspaces. Additionally, we serve as consultants for DBREs of other Business Units (BUs) and the Command Center, working together to define RPOs, RTOs, and SLOs with developers to ensure database reliability.

    We also evaluate financial optimizations by enhancing performance and ensuring system scalability. We integrate monitoring with Dynatrace, CloudWatch, and Prometheus with Grafana to track resources and conduct POCs for services to meet team needs. Additionally, we contribute expertise on various AWS and OCI services, ensuring our infrastructure and automations are robust, secure, and efficient. Our efforts are crucial in maintaining PicPay's agility, innovation, and competitiveness in the financial market by ensuring high availability, performance, and resilience of database systems.

* Principal Reliability Engineer & Tech Lead, Tag IMF - Stone Co.
  * Jun 2023 - Sep 2023
  * Location: São Paulo, Brazil
  
    After demonstrating my expertise and leadership abilities, I was promoted to Principal DBRE and Tech Lead, where I managed a team of five people. In addition to hands-on technical work, my responsibilities expanded to include people management and team development.

    I continued to drive process improvements and automation initiatives, creating CI/CD pipelines for databases using Terraform, Terragrunt, and GitHub Actions. These efforts significantly improved the speed and standardization of database creation for the team. I also led cost reduction initiatives, successfully decreasing high cloud and MongoDB Atlas expenses by improving and modernizing existing services.

    To further reduce costs, I implemented processes to shut down non-essential environments during weekends and off-peak hours. I also created auto-scaling routines for environments with varying usage patterns, ensuring resources were used efficiently based on demand.

    In my managerial role, I provided mentorship and guidance to my team, ensuring they were equipped with the necessary skills and knowledge to excel in their roles. I fostered a collaborative and innovative work environment, encouraging continuous learning and professional growth. My dual focus on technical excellence and team development contributed to the overall efficiency and reliability of our database operations, ensuring robust and cost-effective infrastructure management.

* Staff Database Reliability Engineer, Tag IMF - Stone Co.
  * Oct 2022 - Jun 2023
  * Location: São Paulo, Brazil
  
    I joined the team as a specialist DBRE to enhance processes, automation, and financial efficiency. I created Terraform automations using GitHub Actions pipelines to automate the creation, destruction, and ServiceNow GMUD requests, ensuring adherence to company processes. Additionally, I developed Golang and Python scripts to import manually created resources into Terraform, standardizing and automating infrastructure management.

    My primary focus was on Google Cloud Platform (GCP) and various database technologies, including MongoDB Atlas, CloudSQL, BigTable, and Neo4j. I also conducted proof of concepts (POCs) with CockroachDB, ScyllaDB, and YugabyteDB.

    One significant achievement was optimizing our BigTable cluster, which initially ran with 200 nodes. By improving performance, we reduced the cluster to 25 nodes, significantly lowering costs. Similarly, for our MongoDB clusters, most of which used sharding and housed several terabytes of data, I improved performance and reduced costs. We managed to downsize from an M200 cluster to an M50 and reduced the shard count by three fragments.

* Senior Database Reliability Engineer, iFood
  * Mar 2022 - Oct 2022
  * Location: São Paulo, Brazil
  
    As a Database Reliability Engineer (DBRE) on the PostgreSQL team, I provided support with incident management and performance optimization. I initiated the creation of Proofs of Concept (POCs) using Patroni and developed automation for architecture deployment on AWS with Terraform, Chef, and Ruby, ensuring reliable environments on virtual machines. I expanded my expertise in key AWS services like Auto Scaling Groups (ASG), CloudWatch, EC2, and Lambda, while improving observability with Datadog. Additionally, I worked with ScyllaDB, a high-performance columnar database, and deepened my understanding of Kafka and Golang.

    I also leveraged tools such as PgBouncer for connection pooling and PgBackRest for reliable backups, ensuring data integrity and availability. To ensure high availability during failovers, I utilized HAProxy and AWS Elastic Load Balancing (ELB). I assisted development teams by defining reliability engineering best practices, including automated failover mechanisms, optimizing database performance, ensuring data consistency, and establishing robust monitoring and alerting standards. These efforts were crucial in maintaining high availability and resilience of our database systems, ultimately supporting the overall reliability and performance of our applications.

* Senior Database Reliability Engineer, Stone Co.
  * Mar 2021 - Mar 2022
  * Location: Rio de Janeiro, Brazil
  
    I worked as a Database Reliability Engineer (DBRE) in the transactional tribe, focusing on the transactional BU that used technologies like SQL Server, PostgreSQL on Patroni, and MongoDB in a hybrid environment, both on-premises and in Azure Cloud. I created hybrid architectures, leveraging the strengths of both on-premise and cloud environments to build scalable and reliable systems.

    For PostgreSQL, I utilized tools such as PgBouncer for connection pooling, PgBackRest for reliable backups, and Patroni, HAProxy, etcd, and keepalived to ensure high availability and failover capabilities. Additionally, I enabled auditing with pgAudit and optimized virtual machine parameters for peak performance.

    For MongoDB, I implemented Percona Backup for MongoDB (PBM) for backups, used MongoDB Percona with TLS/SSL and LDAP for secure authentication, and enabled audit filters. We also adopted MongoDB Atlas to enhance our hybrid database solutions.

    I automated database deployment using Azure Pipelines, Terraform, and Ansible, and created proof of concept (POC) projects for new technologies, developing and maintaining automation scripts for architecture deployment and reliability.

    I supported other BUs and group companies by assessing performance and handling incidents, gaining experience with MongoDB Atlas, Google Cloud Platform, and AWS. I contributed to the team's development by mentoring junior members, leading training sessions, and sharing knowledge to ensure the team was equipped with the latest database management practices and technologies.

    I implemented monitoring solutions using Prometheus, New Relic, and Grafana, integrating these with PagerDuty for alerting and incident management.

    My role involved communicating with stakeholders to understand requirements and provide technical guidance, fostering a collaborative environment. By encouraging teamwork and continuous learning, I helped create a culture of growth and innovation within the team.

* Database Reliability Engineer, Stone Co.
  * Dec 2020 - Mar 2021
  * Location: Rio de Janeiro, Brazil
  
    I worked as a Database Reliability Engineer (DBRE) in the transactional tribe, focusing on the transactional BU that used technologies like SQL Server, PostgreSQL on Patroni, and MongoDB in a hybrid environment, both on-premises and in Azure Cloud. I created hybrid architectures, leveraging the strengths of both on-premise and cloud environments to build scalable and reliable systems.

    For PostgreSQL, I utilized tools such as PgBouncer for connection pooling, PgBackRest for reliable backups, and Patroni, HAProxy, etcd, and keepalived to ensure high availability and failover capabilities. Additionally, I enabled auditing with pgAudit and optimized virtual machine parameters for peak performance.

    For MongoDB, I implemented Percona Backup for MongoDB (PBM) for backups, used MongoDB Percona with TLS/SSL and LDAP for secure authentication, and enabled audit filters. We also adopted MongoDB Atlas to enhance our hybrid database solutions.

    I automated database deployment using Azure Pipelines, Terraform, and Ansible, and created proof of concept (POC) projects for new technologies, developing and maintaining automation scripts for architecture deployment and reliability.

    I supported other BUs and group companies by assessing performance and handling incidents, gaining experience with MongoDB Atlas, Google Cloud Platform, and AWS. I contributed to the team's development by mentoring junior members, leading training sessions, and sharing knowledge to ensure the team was equipped with the latest database management practices and technologies.

    I implemented monitoring solutions using Prometheus, New Relic, and Grafana, integrating these with PagerDuty for alerting and incident management.

    My role involved communicating with stakeholders to understand requirements and provide technical guidance, fostering a collaborative environment. By encouraging teamwork and continuous learning, I helped create a culture of growth and innovation within the team.

* Infrastructure Jr. Analyst, Getnet Brasil
  * Aug 2020 - Dec 2020
  * Location: Porto Alegre, Rio Grande do Sul, Brazil
  
    As a member of the N1 Database team, I was responsible for handling initial demands and incidents across various database technologies in a completely local environment. Our technologies included Oracle Data Guard, RAC, Goldengate, ASM and RMAN, as well as SQL Server with Mirror and Always On, MySQL, Sybase, MariaDB and MongoDB.

    In addition to database management, I supported the infrastructure, backup and middleware teams. I worked with technologies such as TSM for backups, WebLogic, OHS, OSB and SOA for middleware, and assisted the batch processing team using Control-M and Connect Direct. I also used Ansible and Ansible Tower to automate several processes, ensuring reliability, standardization and efficiency. For example, I helped automate PSU patching in Oracle databases.

    I was a self-taught professional who continually sought to expand my knowledge and skills. I shared my experience with colleagues, fostering a culture of learning and continuous improvement. My cross-functional role has allowed me to develop a comprehensive understanding of the entire IT ecosystem, ensuring seamless operation and integration between various systems and technologies. This collaborative and innovative approach has allowed me to significantly contribute to the reliability and efficiency of our database and IT infrastructure.

* Database Jr. Analyst, Getnet Brasil
  * Nov 2019 - Jul 2020
  * Location: Campo Bom, Rio Grande do Sul, Brazil
  
    As a member of the N1 Database team, I was responsible for handling initial demands and incidents across various database technologies in a completely local environment. Our technologies included Oracle Data Guard, RAC, Goldengate, ASM and RMAN, as well as SQL Server with Mirror and Always On, MySQL, Sybase, MariaDB and MongoDB.

    In addition to database management, I supported the infrastructure, backup and middleware teams. I worked with technologies such as TSM for backups, WebLogic, OHS, OSB and SOA for middleware, and assisted the batch processing team using Control-M and Connect Direct. I also used Ansible and Ansible Tower to automate several processes, ensuring reliability, standardization and efficiency. For example, I helped automate PSU patching in Oracle databases.

    I was a self-taught professional who continually sought to expand my knowledge and skills. I shared my experience with colleagues, fostering a culture of learning and continuous improvement. My cross-functional role has allowed me to develop a comprehensive understanding of the entire IT ecosystem, ensuring seamless operation and integration between various systems and technologies. This collaborative and innovative approach has allowed me to significantly contribute to the reliability and efficiency of our database and IT infrastructure.

* Monitoring Operator, Getnet Brasil
  * Sep 2017 - Oct 2019
  * Location: Campo Bom, Rio Grande do Sul, Brazil
  
    As part of the NOC/Operation team, I was responsible for monitoring Production, Homologation, Development, and Transactional environments. I acted as the first level of support for incidents impacting clients, the call center, and various sectors of the company. My role involved ensuring rapid response and resolution to maintain system stability and customer satisfaction.

    In addition to incident management, I was in charge of automating processes using the Control-M tool and monitoring these automated jobs. This involved creating, scheduling, and managing jobs to ensure efficient workflow and system performance. I also developed automations using Shell Script, PHP, and other technologies to streamline and enhance critical processes for the team.

    Furthermore, I created automations using Ansible to install Control-M in a standardized and automated manner, ensuring consistency and reliability across all deployments.

    By leveraging my skills in automation and incident management, I contributed to the smooth operation of critical business processes and supported the overall reliability of our IT infrastructure. My proactive approach to automation helped minimize downtime and improve operational efficiency. My role was crucial in maintaining high availability and performance across all environments, ensuring that our systems met the needs of both internal and external stakeholders.

* Call Center Assistant, Getnet Brasil
  * Jan 2017 - Aug 2017
  * Location: Campo Bom, Rio Grande do Sul, Brazil
    
      As a Call Center Assistant, I provided support to clients and internal teams, handling inquiries, complaints, and technical issues. I was responsible for ensuring customer satisfaction by addressing their concerns promptly and effectively.
  
      I also assisted in the implementation of new services and products, providing guidance and support to clients during the onboarding process. I worked closely with the sales and marketing teams to ensure a seamless customer experience and promote the company's products and services.
  
      My role required strong communication and problem-solving skills, as well as the ability to work under pressure and meet tight deadlines. I developed a deep understanding of our products and services, enabling me to provide accurate and timely assistance to clients. My dedication to customer service and commitment to excellence helped drive customer satisfaction and loyalty, contributing to the overall success of the company.

Publications
======
  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
  
Talks
======
  <ul>{% for post in site.talks reversed %}
    {% include archive-single-talk-cv.html  %}
  {% endfor %}</ul>
  
Badges
======
  {% include badges.html %}

Certifications
======
  <ul>{% for post in site.certifications reversed %}
    {% include archive-single-certification-cv.html  %}
  {% endfor %}</ul>
  
Service and leadership
======
* Community Creator on MongoDB
* Subject Matter Expert on MongoDB - Atlas Administration
* Lead Subject Matter Expert on MongoDB - Database Administration
* Top Database Administration Voice on Linkedin
* Programmer Professional Qualification Certificate - Unisinos
* Highlight of the IT Infrasctructure team 2019 - Getnet Brasil