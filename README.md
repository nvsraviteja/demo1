#!/bin/bash

# GitHub repository details
REPO_OWNER="your-username-or-organization"
REPO_NAME="your-repo-name"
GITHUB_TOKEN="your-github-token"

# File to store output
OUTPUT_FILE="reviewed_prs.txt"

# Remove the output file if it exists
[ -f $OUTPUT_FILE ] && rm $OUTPUT_FILE

# Fetch the last 100 PRs
echo "Fetching the last 100 pull requests..."
prs=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=100")

# Loop through each PR by extracting the PR numbers using grep and sed
echo "Processing pull requests..."
for pr_number in $(echo "$prs" | grep '"number":' | awk '{print $2}' | sed 's/,//'); do
    # Fetch reviews for the PR
    reviews=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
        "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls/$pr_number/reviews")

    # Check if the PR has any reviews
    if echo "$reviews" | grep -q '"id":'; then
        # Extract the PR title and URL using grep, awk, and sed
        pr_title=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"title":' | awk -F '"' '{print $4}')
        pr_url=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"html_url":' | awk -F '"' '{print $4}')

        # Write the PR details to the output file
        echo "PR #$pr_number: $pr_title" >> "$OUTPUT_FILE"
        echo "URL: $pr_url" >> "$OUTPUT_FILE"
        echo "" >> "$OUTPUT_FILE"
    fi
done

echo "Reviewed pull requests saved to $OUTPUT_FILE"
