# BPO Agent Performance & Attrition Analytics Platform [40% Done; Discontinued]
## Project Retrospective & Learning Documentation

**Project Status:** Discontinued  
**Tech Stack:** Azure Data Factory, Synapse Analytics Serverless, ADLS Gen2, Power BI

---

## Project Overview

**Objective:** Build an enterprise-grade analytics platform for BPO operations using Azure cloud services, implementing medallion architecture (Bronze/Silver/Gold) for processing agent performance and attrition data.

**Planned Features:**
- Automated batch data ingestion pipelines
- Data quality validation and cleansing
- Business-ready analytics layer
- Interactive dashboards
- Data governance and lineage

**Target Metrics:**
- 500+ agent records
- 2000+ performance logs
- Multi-source data integration (CSV, JSON)
- Real-time query performance on historical data

---

## Milestones Achieved

### Infrastructure Setup (Completed)
- Azure resource provisioning (Resource Group, ADLS Gen2, ADF, Synapse)
- Storage account with medallion architecture (Bronze/Silver/Gold containers)
- Synthetic data generation (500 agents, 2000+ performance logs, realistic BPO metrics)
- Data uploaded to ADLS raw layer
- GitHub repository initialized

**Key Learning:** Free tier limitations require careful service selection (serverless over dedicated pools)

---

### Data Ingestion (Partially Completed)
**Completed:**
- ADF linked services (ADLS Gen2, Synapse serverless)
- 8 datasets created (4 source CSV/JSON, 4 sink Parquet)
- Copy Data pipeline built with Get Metadata validation
- Parquet files successfully generated in Bronze layer (23 KB+ with data)
- Master orchestration pipeline design

**Blocked:**
- External table creation in Synapse
- Query execution on Bronze layer
- Watermark table implementation (CREATE TABLE not supported in serverless)

---

## Major Roadblocks

### 1. **Synapse Serverless Limitations**
**Issue:** Cannot create regular tables for metadata (watermark tracking)  
**Impact:** Unable to implement incremental load pattern  
**Attempted Solutions:**
- Tried CREATE TABLE syntax (not supported)
- Explored external tables (different purpose)
- Considered Azure SQL Database alternative (adds cost)

**Learning:** Synapse Serverless is query-only; metadata requires separate storage or alternative patterns

---

### 2. **Authentication & Permissions Complexity**
**Issue:** Multiple authentication failures across services  
**Root Causes:**
- Azure AD-only authentication enabled (blocks SQL logins)
- Managed Identity not registered in Azure AD
- Missing Storage Blob Data Contributor roles
- Credential syntax variations across Azure services

**Attempted Solutions (15+ iterations):**
- SQL authentication (blocked by AD-only mode)
- System Assigned Managed Identity (principal not found)
- Storage Account Keys (credential syntax errors)
- SAS Tokens (access still denied)
- Database Scoped Credentials (master key required)
- Object ID instead of name (not recognized)
- Public blob access test (confirmed permissions issue)

**Error Messages Encountered:**
```
"Login failed for user '<token-identified principal>'"
"Principal 'adf-bpo-analytics-prod' could not be found"
"External table is not accessible because content cannot be listed"
"Please create a master key in the database"
"Azure Active Directory only authentication is enabled"
"Cannot drop credential because it is used by external data source"
```

**Learning:** Azure authentication requires precise IAM role assignments, AD registration, and 2-5 minute propagation delays

---

### 3. **External Table Query Returns Empty Results**
**Issue:** Table structure created, columns visible, but SELECT returns no data (all NULL)  
**File Verification:**
- Parquet file exists: `bronze/agents/agents.parquet`
- File size: 23.17 KiB (confirmed data present)
- ADF pipeline: Succeeded, 505 rows copied

**Root Cause:** User account lacks Storage Blob Data Contributor role on ADLS  
**Status:** Attempted to grant role via IAM, but authentication complexity prevented verification


---

### 4. **Microsoft Purview Registration Failure**
**Issue:** "Could not create the marketplace item"  
**Root Cause:** Purview not available in free tier/student accounts or regional restrictions  
**Workaround:** Governance redesigned using SQL metadata tables + Excel catalog  
**Impact:** Eliminated $15/month cost, simplified architecture

---

### 5. **Synapse SQL Syntax Limitations**
**Issues:**
- `DROP IF EXISTS` not supported
- `CREATE TABLE` not supported (serverless)
- Multi-line credential syntax errors
- Master key requirement unclear

**Learning:** Synapse Serverless SQL has different syntax from standard SQL Server

---

## Technical Decisions & Trade-offs

### Kept Decisions
- **Synapse Serverless over Dedicated Pool:** $0 vs. $$$, suitable for MVP
- **Parquet over CSV in Bronze:** Columnar format, compression, query performance
- **Manual governance over Purview:** Cost savings, sufficient for demo project
- **ADF over manual scripts:** Enterprise pattern, monitoring, scheduling

###  Abandoned Features
- **Watermark table:** Requires Azure SQL DB (adds cost) or file-based alternative
- **Purview governance:** Unavailable in subscription tier
- **Managed Identity auth:** Registration complexity vs. timeline
- **Real-time incremental loads:** Simplified to full refresh pattern

---

## Key Learnings

### **Azure Free Tier Constraints**
- Purview unavailable in some subscriptions
- Synapse Serverless: Query-only, no INSERT/UPDATE
- Managed Identity registration requires specific permissions
- Role propagation: 2-5 minutes minimum, sometimes longer

### **Authentication Best Practices**
- Start with simplest auth method (SQL) before managed identity
- Verify IAM roles in Portal → IAM → Role assignments tab
- Check both workspace-level and resource-level permissions
- Azure AD rename (now Entra ID) causes documentation confusion

### **Troubleshooting Approach**
1. Verify resource names exactly (case-sensitive, hyphen-sensitive)
2. Check file existence and size in Portal before querying
3. Test with OPENROWSET before creating external tables
4. Separate concerns: ADF auth ≠ User auth in Synapse Studio
5. Enable public access temporarily to isolate permission issues

### **Data Engineering Patterns**
- Medallion architecture works well with ADLS + Synapse
- External tables = read-only view over files (not data storage)
- ADF + Synapse Serverless compatible despite authentication issues
- Parquet file generation succeeded even when queries failed

---

## Project Metrics

**Troubleshooting Breakdown:**
- Authentication issues: 60% of time
- Syntax/configuration: 25%
- Service limitations research: 15%

**Azure Resources Created:**
- 1 Resource Group
- 1 Storage Account (ADLS Gen2)
- 1 Data Factory V2
- 1 Synapse Analytics Workspace
- 4 containers (raw, bronze, silver, gold)
- 8 ADF datasets
- 4 ADF pipelines (Copy + Master orchestration)

**Files Generated:**
- 4 CSV/JSON source files (500+ rows each)
- 4 Parquet files in Bronze layer (~20-30 KB each)
- 1 Python data generation script
- ADF pipeline JSON exports

---

## Why Project Was Discontinued

**Primary Blocker:** Authentication complexity exceeded project timeline value  
**Contributing Factors:**
- Free tier limitations preventing managed identity setup
- Azure AD-only mode blocking SQL authentication fallback
- 15+ authentication iterations without resolution
- Synapse Serverless constraints requiring architecture redesign

**Decision Point:** After 10 hours troubleshooting, authentication issues blocked core functionality (querying data). Continuing would require:
- Azure subscription with full permissions
- Or 4+ more hours for SQL Database + alternative auth setup
- Or complete architecture pivot away from Synapse

**ROI Assessment:** Learning value (authentication, Azure services) achieved; remaining time better spent on new projects with clearer success path

---

## Recommendations for Future Attempts

### **If Retrying This Project:**
1. **Use Azure SQL Database instead of Synapse Serverless** for Bronze/Silver layers
   - Cost: ~$5-10/month serverless tier
   - Benefit: Full SQL support (INSERT, UPDATE, tables)
   - Trade-off: Not as "cloud-native" but more functional

2. **Start with SQL Authentication** before attempting managed identity
   - Disable Azure AD-only mode from day 1
   - Add managed identity later as optimization

3. **Verify subscription permissions upfront:**
   - Can create managed identities?
   - Can register external providers?
   - Purview available?

4. **Simplify architecture for MVP:**
   - Skip watermark table (full refresh acceptable for demo)
   - Skip Purview (document manually)
   - Reduce authentication touchpoints

5. **Use Azure Databricks instead:**
   - Unity Catalog for governance (free)
   - Delta Lake tables (read/write capable)
   - Notebooks avoid ADF complexity
   - Better free tier support

### **Alternative Stack (Proven):**
- **Data Lake:** ADLS Gen2 ✓
- **Processing:** Databricks Community Edition (free)
- **Orchestration:** Databricks Workflows or ADF
- **Governance:** Unity Catalog (free, included)
- **Visualization:** Power BI Free

---

## Skills Demonstrated

**Despite incomplete status, project demonstrated:**

### **Azure Services:**
- Resource provisioning and management
- ADLS Gen2 container and folder structure
- Azure Data Factory pipeline development
- Synapse Analytics workspace configuration
- IAM role assignment attempts

### **Data Engineering:**
- Medallion architecture design (Bronze/Silver/Gold)
- Data generation with realistic business logic
- ETL pipeline design (Extract from CSV → Transform to Parquet → Load to Bronze)
- Parquet file format usage
- Data quality considerations (nulls, duplicates, outliers)

### **Problem Solving:**
- Systematic troubleshooting (15+ authentication approaches)
- Documentation review (Azure docs, Stack Overflow, error messages)
- Iterative testing (OPENROWSET, external tables, credentials)
- Root cause analysis (permission vs. syntax vs. service limitations)
- Knowing when to pivot vs. persist

### **Professional Skills:**
- Cost optimization awareness (free tier selection)
- Security considerations (least privilege, credential types)
- Project scoping and timeline management
- Technical documentation and retrospectives

---

## Conclusion

This project successfully demonstrated Azure service integration and data engineering patterns. While authentication complexity blocked final delivery, the experience provided valuable learning in:

- Azure cloud architecture
- Authentication and authorization in enterprise environments
- Service limitation trade-offs
- When to pivot vs. troubleshoot
- Production-ready documentation

The medallion architecture, data generation, and ADF pipeline work are reusable for future projects. Authentication lessons learned will accelerate similar Azure implementations.

**Key Takeaway:** Sometimes the most valuable learning comes from understanding what *doesn't* work and why, not just successful completions.

---

## Repository Structure

```
bpo-analytics-platform/
├── data-generation/
│   └── generate_bpo_data.py (synthetic data script)
├── docs/
│   ├── architecture-diagram.png (planned)
│   └── project-retrospective.md (this document)
├── adf-pipelines/
│   └── (ADF ARM templates - exported)
└── README.md
```

**GitHub Status:** Architecture and lessons documented  
**LinkedIn Summary:** "Built BPO analytics pipeline with Azure Data Factory and Synapse, demonstrating cloud data engineering patterns and troubleshooting complex authentication scenarios in enterprise environments."

---

**Total Azure Cost:** ~$0.50 (storage only, serverless compute)  
**Learning Value:** High (cloud services, authentication, trade-offs)  
**Completion:** 40% 
**Recommendation:** Valuable as learning project; production requires different architecture or subscription tier

---

*Document Version: 1.0*  
*Last Updated: February 2025*  
*Author: Analytics Engineering Portfolio Project*
