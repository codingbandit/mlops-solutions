# Challenge 4: Work with linting and unit testing

## Fix code checks action/workflow

On the Actions page, run the workflow for **Code Checks**.

The action run will fail the linter. Fix train.py linting errors.

Final linted source file:

```python
# Import libraries
import mlflow
import argparse
import glob
import os

import pandas as pd

from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split


# define functions
def main(args):
    # TO DO: enable autologging
    mlflow.autolog()

    # read data
    df = get_csvs_df(args.training_data)

    # split data
    X_train, X_test, y_train, y_test = split_data(df)

    # train model
    train_model(args.reg_rate, X_train, X_test, y_train, y_test)


def get_csvs_df(path):
    if not os.path.exists(path):
        raise RuntimeError(f"Cannot use non-existent path provided: {path}")
    csv_files = glob.glob(f"{path}/*.csv")
    if not csv_files:
        raise RuntimeError(f"No CSV files found in provided data path: {path}")
    return pd.concat((pd.read_csv(f) for f in csv_files), sort=False)


# TO DO: add function to split data
def split_data(df):
    features_array = [
        'Pregnancies', 'PlasmaGlucose',
        'DiastolicBloodPressure', 'TricepsThickness',
        'SerumInsulin', 'BMI',
        'DiabetesPedigree', 'Age'
        ]

    X, y = df[features_array].values, df['Diabetic'].values

    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.30,
                                                        random_state=0)
    return X_train, X_test, y_train, y_test


def train_model(reg_rate, X_train, X_test, y_train, y_test):
    # train model
    LogisticRegression(C=1/reg_rate, solver="liblinear").fit(X_train, y_train)


def parse_args():
    # setup arg parser
    parser = argparse.ArgumentParser()

    # add arguments
    parser.add_argument("--training_data", dest='training_data',
                        type=str)
    parser.add_argument("--reg_rate", dest='reg_rate',
                        type=float, default=0.01)

    # parse args
    args = parser.parse_args()

    # return args
    return args


# run script
if __name__ == "__main__":
    # add space in logs
    print("\n\n")
    print("*" * 60)

    # parse args
    args = parse_args()

    # run main function
    main(args)

    # add space in logs
    print("*" * 60)
    print("\n\n")

```

## Create new YAML file and define workflow to do code checks on each pull request

Create new YAML file `04-code-checks-pull.yml` in the **.github\workflows** folder.

```yml
name: Code checks on pull

on:
  pull_request:

jobs:
  LinterJob:
    name: linting
    runs-on: ubuntu-latest
    steps:
    - name: Check out repo
      uses: actions/checkout@main
    - name: Use Python version 3.8
      uses: actions/setup-python@v3
      with:
        python-version: '3.8'
    - name: Install Flake8
      run: |
        python -m pip install flake8
    - name: Run linting tests
      run: | 
        flake8 src/model/
  UnitTestJob:
    name: unit tests
    runs-on: ubuntu-latest
    steps:
    - name: Check out repo
      uses: actions/checkout@main
    - name: Use Python version 3.8
      uses: actions/setup-python@v3
      with:
        python-version: '3.8'
    - name: Install Python dependencies
      uses: py-actions/py-dependency-install@v4
      with:
        path: "requirements.txt"
    - name: Run tests
      run: |
        pytest tests/
```

## Add branch protection rule to require linting and unit tests every pull request

The **Search** box for status checks are looking at the names of the jobs in the YAML file. From above, we named the jobs: `linting` and `unit tests`.

1. On the GitHub repo page, select **Settings**
2. Select **Branches** (Beneath Code and automation heading).
3. Select **Add rule**.
4. For branch name pattern, enter `*/*`, this will match **feature/featurenmame** branches.
5. Check the **Require status checks to pass before merging** for the branch protection rule. Also check the **Require branches to be up to date before merging** checkbox.
6. Search for and select **linting** for the status check.
7. Search for and select **unit tests** for a second status check.
8. Select **Create**.
