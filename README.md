# Data Processing & CI/CD Pipeline

This repository hosts a data processing application designed to transform `data.xlsx` into aggregated JSON output using Python with Pandas, fully integrated with a GitHub Actions CI/CD pipeline for automated testing and deployment.

## Project Overview

The core of this project involves:
1.  **Data Conversion**: Converting the source `data.xlsx` file into `data.csv` for easier programmatic access and processing.
2.  **Data Processing**: Executing `execute.py` which reads `data.csv`, performs specific aggregations (e.g., summing values by category), and outputs the results to `result.json`.
3.  **Automated Workflow**: A GitHub Actions workflow (`.github/workflows/ci.yml`) that automates code quality checks (using Ruff), executes the data processing script, and publishes the `result.json` file via GitHub Pages.

## Getting Started

### Prerequisites

To run this project locally, ensure you have:
*   Python 3.11+
*   Pandas 2.3+
*   Ruff (for linting)
*   Openpyxl (to read `.xlsx` files if converting manually)

You can install the required Python packages using pip:
```bash
pip install pandas==2.3.0 ruff openpyxl
```

### Local Development

1.  **Clone the Repository**:
    ```bash
git clone https://github.com/your-username/your-repo-name.git
cd your-repo-name
    ```
2.  **Prepare Data**:
    First, convert `data.xlsx` to `data.csv`. This step is automated in CI, but for local execution, you can do it manually:
    ```bash
python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
    ```
    Ensure `data.xlsx` is present in the root directory.
3.  **Run the Processor**:
    Execute the main Python script to generate `result.json`:
    ```bash
python execute.py > result.json
    ```
    This command will run `execute.py` which processes `data.csv` and outputs the aggregated data to `result.json`.
4.  **Linting**:
    Check code quality with Ruff:
    ```bash
ruff check .
    ```

## Files in this Repository (Conceptual Contents)

While this output focuses on providing the core documentation and HTML, the project conceptually includes the following key files. Their full content is embedded here to satisfy the request of providing them.

### `execute.py` (Fixed and Enhanced)

This script is responsible for reading the processed CSV data, performing aggregations, and outputting the results as JSON. It has been fixed to include robust error handling, version checks, and proper data type conversions.

```python
import pandas as pd
import sys
import json
import os

# Ensure Python 3.11+ is used
if sys.version_info < (3, 11):
    print("Error: Python 3.11 or higher is required.", file=sys.stderr)
    sys.exit(1)

def check_pandas_version():
    """Checks if Pandas version is 2.3 or higher."""
    pd_version_info = tuple(map(int, pd.__version__.split('.')))
    if pd_version_info < (2, 3):
        print(f"Error: Pandas 2.3 or higher is required. Found: {pd.__version__}", file=sys.stderr)
        sys.exit(1)

def process_data(input_csv_path="data.csv", output_json_path="result.json"):
    """
    Reads data from CSV, performs aggregation, and saves results to JSON.
    This function includes checks for required columns, handles non-numeric
    values, and performs a sum aggregation.
    """
    check_pandas_version() # Ensure Pandas version is correct before processing

    if not os.path.exists(input_csv_path):
        print(f"Error: The input CSV file '{input_csv_path}' was not found.", file=sys.stderr)
        sys.exit(1)

    try:
        df = pd.read_csv(input_csv_path)

        # Expected columns. This is a common point for "non-trivial errors" if not handled.
        expected_columns = ['Category', 'Value']
        if not all(col in df.columns for col in expected_columns):
            missing_cols = [col for col in expected_columns if col not in df.columns]
            print(f"Error: Input CSV is missing required columns: {', '.join(missing_cols)}.", file=sys.stderr)
            sys.exit(1)

        # Convert 'Value' to numeric, coercing errors will turn non-numeric into NaN.
        # This is another common source of errors if data is dirty.
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')

        # Fill NaN values in 'Value' column with 0 before aggregation.
        # This "fix" prevents NaN values from propagating or causing issues in sum.
        df['Value'].fillna(0, inplace=True)

        # Perform aggregation: group by 'Category' and sum 'Value'
        result_df = df.groupby('Category')['Value'].sum().reset_index()
        result_df.rename(columns={'Value': 'TotalValue'}, inplace=True)

        # Save the aggregated data to a JSON file
        # Using orient="records" for a list of dictionaries, suitable for many APIs
        result_df.to_json(output_json_path, orient="records", indent=4)
        print(f"Data successfully processed and results saved to '{output_json_path}'.")

    except pd.errors.EmptyDataError:
        print(f"Error: The input CSV file '{input_csv_path}' is empty.", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"An unexpected error occurred during data processing: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    process_data()
```

### `.github/workflows/ci.yml` (GitHub Actions Workflow)

This workflow automates linting, data processing, and deployment to GitHub Pages.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - master

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: "3.11"
        cache: 'pip'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pandas==2.3.0 ruff openpyxl

    - name: Run Ruff Linter
      run: ruff check .
      # Optional: Add --fix and run ruff format if auto-fixing is desired
      # run: |
      #   ruff check . --fix
      #   ruff format .
      #   git diff --exit-code # Ensures no uncommitted changes after auto-fixing

    - name: Convert data.xlsx to data.csv
      run: python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
      # Ensure data.xlsx exists before this step, otherwise it will fail.

    - name: Execute data processing script
      run: python execute.py > result.json

    - name: Upload result.json as artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: 'result.json'

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

## GitHub Actions CI/CD Pipeline (`.github/workflows/ci.yml`)

The CI/CD pipeline automates the following steps on every push to the `main` branch:

1.  **Setup**: Sets up a Python 3.11 environment and installs `pandas`, `ruff`, and `openpyxl`.
2.  **Data Conversion**: Converts `data.xlsx` to `data.csv`.
3.  **Linting**: Runs `ruff` to ensure code quality and consistency. The results are visible in the CI logs.
4.  **Data Processing**: Executes `python execute.py`. The output is redirected to `result.json`.
5.  **Artifact Upload**: `result.json` is uploaded as a GitHub Pages artifact.
6.  **GitHub Pages Deployment**: The artifact is deployed to GitHub Pages, making `result.json` publicly accessible (e.g., at `https://your-username.github.io/your-repo-name/result.json`).

The `result.json` file is *not* committed to the repository; it is dynamically generated and published by the CI pipeline.

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.