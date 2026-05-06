# CodeQL SARIF Generation Guide for OWASP Benchmark

This guide documents the full process of downloading CodeQL, creating a CodeQL database for the OWASP Benchmark (Version 1.2), running the analysis for 10 official CWE categories, and preparing the resulting SARIF files for LLM analysis.

---

## STEP 1: CODEQL SETUP (ONE-TIME)

### 1.1. Download CodeQL CLI

```bash
# Create setup directory
mkdir -p ~/codeql-setup
cd ~/codeql-setup

# Download CodeQL CLI (Linux x64)
wget https://github.com/github/codeql-cli-binaries/releases/latest/download/codeql-linux64.zip

# Extract
unzip codeql-linux64.zip

# Move to home directory
mv codeql ~/codeql

# Add to PATH
echo 'export PATH=$PATH:$HOME/codeql' >> ~/.bashrc
source ~/.bashrc

# Verify installation
codeql --version
```

**Expected output:**
```text
CodeQL command-line toolchain release 2.x.x
```

### 1.2. Clone CodeQL Standard Libraries & Queries

```bash
cd ~/codeql-setup

# Clone CodeQL repo (contains all queries)
git clone https://github.com/github/codeql.git codeql-repo

# Move to home directory
mv codeql-repo ~/codeql-repo

# Verify queries are available
ls ~/codeql-repo/java/ql/src/Security/CWE/
```

**Expected output:** Directories like CWE-022, CWE-078, CWE-079, etc.

---

## STEP 2: DOWNLOAD OWASP BENCHMARK SOURCE CODE

### 2.1. Clone OWASP Benchmark

```bash
# Create projects directory
mkdir -p ~/benchmark-projects
cd ~/benchmark-projects

# Clone OWASP Benchmark Java
git clone https://github.com/OWASP-Benchmark/BenchmarkJava.git
cd BenchmarkJava

# Check versions
git tag
# Checkout version 1.2 (corresponds to ground truth 1.2)
git checkout 1.2
```

**OWASP Benchmark Structure:**
```text
BenchmarkJava/
├── src/
│   └── main/
│       └── java/
│           └── org/
│               └── owasp/
│                   └── benchmark/
│                       ├── testcode/        # ← Test cases here
│                       │   ├── BenchmarkTest00001.java
│                       │   ├── BenchmarkTest00002.java
│                       │   └── ... (2742 files)
│                       ├── helpers/
│                       └── service/
├── pom.xml
└── ...
```

### 2.2. Build Project

CodeQL requires a successful project build to analyze Java code.

```bash
cd ~/benchmark-projects/BenchmarkJava

# Install Maven if not available
sudo apt install maven -y  # Ubuntu/Debian
# or
brew install maven         # macOS

# Build project
mvn clean compile -DskipTests

# Verify build success
ls target/classes/org/owasp/benchmark/testcode/
```

---

## STEP 3: CREATE CODEQL DATABASE

### 3.1. Create Database

```bash
# Create database from OWASP Benchmark
codeql database create ~/codeql-db/owasp-benchmark \
  --language=java \
  --source-root=~/benchmark-projects/BenchmarkJava \
  --command="mvn clean compile -DskipTests" \
  --threads=4
```
*Note: This process takes about 5-15 minutes.*

**Expected output:**
```text
Finalizing database at /home/user/codeql-db/owasp-benchmark.
Successfully created database at /home/user/codeql-db/owasp-benchmark.
```

### 3.2. Verify Database

```bash
# Check database info
codeql database info ~/codeql-db/owasp-benchmark
```
**Expected output:**
```text
Language: java
Source location: /home/user/benchmark-projects/BenchmarkJava
```

---

## STEP 4: RUN CODEQL ANALYSIS & GENERATE 10 SARIF FILES

### 4.1. Create output directory

```bash
mkdir -p ~/sarif-output/owasp-benchmark
cd ~/sarif-output/owasp-benchmark
```

### 4.2. Automated Analysis Script (Recommended)

Create `generate_sarifs_official.sh`:

```bash
cat > ~/sarif-output/generate_sarifs_official.sh << 'EOF'
#!/bin/bash

# Database path
DB=~/codeql-db/owasp-benchmark
QUERIES=~/codeql-repo/java/ql/src/Security/CWE
OUTPUT=~/sarif-output/owasp-benchmark

# 10 Official CWE Categories from the thesis requirements
CWES=(
    "CWE-022"  # Path Traversal (271 cases)
    "CWE-078"  # OS Command Injection (251 cases)
    "CWE-079"  # XSS (274 cases)
    "CWE-089"  # SQL Injection (201 cases)
    "CWE-090"  # LDAP Injection (59 cases)
    "CWE-327"  # Broken Crypto (44 cases)
    "CWE-330"  # Weak Random (4 cases)
    "CWE-501"  # Trust Boundary Violation (75 cases)
    "CWE-614"  # Insecure Cookie (4 cases)
    "CWE-643"  # XPath Injection (6 cases)
)

echo "========================================="
echo "CodeQL Analysis for OWASP Benchmark"
echo "10 Official CWE Categories"
echo "========================================="
echo ""

for i in "${!CWES[@]}"; do
    cwe="${CWES[$i]}"
    num=$((i + 1))

    echo "[$num/10] Analyzing $cwe..."

    if [ -d "$QUERIES/$cwe" ]; then
        codeql database analyze "$DB" \
            "$QUERIES/$cwe/" \
            --format=sarif-latest \
            --output="$OUTPUT/owasp-benchmark-$cwe.sarif" \
            --threads=4 \
            --ram=8192

        if [ $? -eq 0 ]; then
            size=$(ls -lh "$OUTPUT/owasp-benchmark-$cwe.sarif" | awk '{print $5}')
            echo "✅ $cwe completed ($size)"
        else
            echo "❌ $cwe failed"
        fi
    else
        echo "⚠️  Query not found: $QUERIES/$cwe"
    fi
    echo ""
done

echo "========================================="
echo "Analysis Complete!"
echo "========================================="
ls -lh "$OUTPUT"/*.sarif
echo ""
echo "Total: $(ls -1 "$OUTPUT"/*.sarif 2>/dev/null | wc -l) SARIF files"
EOF

# Make executable
chmod +x ~/sarif-output/generate_sarifs_official.sh
```

**Run the script:**
```bash
# This will take about 20-45 minutes
~/sarif-output/generate_sarifs_official.sh
```

### 4.3. Manual Analysis (Alternative)

If the script fails, you can run the commands manually:
```bash
DB=~/codeql-db/owasp-benchmark
QUERIES=~/codeql-repo/java/ql/src/Security/CWE
OUTPUT=~/sarif-output/owasp-benchmark

# CWE-022: Path Traversal
codeql database analyze "$DB" "$QUERIES/CWE-022/" \
    --format=sarif-latest \
    --output="$OUTPUT/owasp-benchmark-CWE-022.sarif"

# Continue for the remaining 9 CWEs...
```

---

## STEP 5: COPY SARIF FILES TO MASTER THESIS PROJECT

```bash
# Create destination directory if it doesn't exist
mkdir -p /home/user/Desktop/Projects/tuandev/thesis/master-thesis/OWASP/input_files/sarif_results/owasp-benchmark

# Copy all SARIF files
cp ~/sarif-output/owasp-benchmark/*.sarif \
   /home/user/Desktop/Projects/tuandev/thesis/master-thesis/OWASP/input_files/sarif_results/owasp-benchmark/

# Verify
ls -lh /home/user/Desktop/Projects/tuandev/thesis/master-thesis/OWASP/input_files/sarif_results/owasp-benchmark/
```

**Expected output: 10 files**
```text
owasp-benchmark-CWE-022.sarif
owasp-benchmark-CWE-078.sarif
owasp-benchmark-CWE-079.sarif
owasp-benchmark-CWE-089.sarif
owasp-benchmark-CWE-090.sarif
owasp-benchmark-CWE-327.sarif
owasp-benchmark-CWE-330.sarif
owasp-benchmark-CWE-501.sarif
owasp-benchmark-CWE-614.sarif
owasp-benchmark-CWE-643.sarif
```

---

## STEP 6: VERIFY COMPATIBILITY

### 6.1. Test with 1 SARIF file

```bash
cd /home/user/Desktop/Projects/tuandev/thesis/master-thesis/OWASP

# Test analysis with CWE-089 (SQL Injection)
python analyze_with_llm.py \
    --sarif_file input_files/sarif_results/owasp-benchmark/owasp-benchmark-CWE-089.sarif \
    --project_src_root ~/benchmark-projects/BenchmarkJava \
    --expected_results_csv input_files/ground_truth/expectedresults-1.2.csv \
    --model "openai/gpt-4o-mini" \
    --prompt_version "optimized" \
    --run_id "test_sarif_compatibility"
```
**If successful:** Creates the `results/optimized_owasp_CWE-089_openai_gpt-4o-mini/` folder with prompts & responses.

### 6.2. Check SARIF Structure

```bash
# Check if a SARIF file holds the correct format
python3 -c "
import json
with open('input_files/sarif_results/owasp-benchmark/owasp-benchmark-CWE-089.sarif') as f:
    data = json.load(f)
    print(f'Results count: {len(data[\"runs\"][0][\"results\"])}')
    print(f'Has codeFlows: {\"codeFlows\" in data[\"runs\"][0][\"results\"][0]}')
"
```

---

## SUMMARY: FINAL FILE STRUCTURE

```text
master-thesis/
├── OWASP/
│   └── input_files/
│       ├── sarif_results/
│       │   └── owasp-benchmark/
│       │       ├── owasp-benchmark-CWE-022.sarif  ✅ (271 test cases)
│       │       ├── owasp-benchmark-CWE-078.sarif  ✅ (251 test cases)
│       │       ├── owasp-benchmark-CWE-079.sarif  ✅ (274 test cases)
│       │       ├── owasp-benchmark-CWE-089.sarif  ✅ (201 test cases)
│       │       ├── owasp-benchmark-CWE-090.sarif  ✅ (59 test cases)
│       │       ├── owasp-benchmark-CWE-327.sarif  ✅ (44 test cases)
│       │       ├── owasp-benchmark-CWE-330.sarif  ✅ (4 test cases)
│       │       ├── owasp-benchmark-CWE-501.sarif  ✅ (75 test cases)
│       │       ├── owasp-benchmark-CWE-614.sarif  ✅ (4 test cases)
│       │       └── owasp-benchmark-CWE-643.sarif  ✅ (6 test cases)
│       ├── ground_truth/
│       │   └── expectedresults-1.2.csv
│       └── prompt_templates/
│
└── OpenVuln/
    └── sarif-files/                           (PRE-EXISTING ✅)
        ├── apache__jspwiki_CVE-2022-46907_2.11.3.sarif
        ├── hapifhir__org.hl7.fhir.core_CVE-2023-28465_5.6.105.sarif
        ├── jeremylong__DependencyCheck_CVE-2018-12036_3.1.2.sarif
        ├── keycloak__keycloak_CVE-2022-4361_21.1.1.sarif
        ├── perwendel__spark_CVE-2016-9177_2.5.1.sarif
        ├── undertow-io__undertow_CVE-2014-7816_1.0.16.Final.sarif
        └── zeroturnaround__zt-zip_CVE-2018-1002201_1.12.sarif
```
*(Total Output: 10 files)*

---

## ESTIMATED TIME

| Step                      | Estimated Time |
| ------------------------- | -------------- |
| 1. Install CodeQL         | 10-15 mins     |
| 2. Download Benchmark     | 5 mins         |
| 3. Create Database        | 10-20 mins     |
| 4. **Analyze 10 CWEs**    | **20-45 mins** |
| 5. Copy files             | 1 min          |
| 6. Verify                 | 5 mins         |
| **TOTAL**                 | **~45-90 mins**|

---

## RUN FULL LLM ANALYSIS

```bash
cd /home/user/Desktop/Projects/tuandev/thesis/master-thesis/OWASP

# Set API keys based on your provider
export OPENROUTER_API_KEY="your-openrouter-key"
export GROQ_API_KEY="your-groq-key"

# Run parallel analysis for all 10 standard CWEs
# (Replace model with your choice, e.g., cliproxy:gemini-2.5-pro for CLI Proxy)
python run_parallel_analysis.py \
    --datasets owasp \
    --prompt-versions optimized \
    --models "cliproxy:gemini-2.5-pro" \
    --cwes CWE-022 CWE-078 CWE-079 CWE-089 CWE-090 CWE-327 CWE-330 CWE-501 CWE-614 CWE-643 \
    --threads 4
```
