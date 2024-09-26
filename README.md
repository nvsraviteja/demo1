#!/bin/bash

# GitHub repository details
REPO_OWNER="your-username-or-organization"
REPO_NAME="your-repo-name"
GITHUB_TOKEN="your-github-token"

# List of usernames to filter by (teammates)
USERNAMES=("user1" "user2" "user3")

# Output CSV file
output_file="pull_requests.csv"

# Check if the file exists; if not, write the CSV header
if [[ ! -f "$output_file" ]]; then
    echo "PR Number,Title,Author,URL,Created Date" > "$output_file"
fi

# Fetch the last 100 PRs
echo "Fetching the last 100 pull requests..."
prs=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=100")

# Loop through each PR using jq to extract necessary fields
echo "Processing pull requests..."
echo "$prs" | jq -c '.[]' | while read -r pr; do
    pr_number=$(echo "$pr" | jq '.number')
    pr_author=$(echo "$pr" | jq -r '.user.login')
    pr_title=$(echo "$pr" | jq -r '.title')
    pr_url=$(echo "$pr" | jq -r '.html_url')
    pr_created=$(echo "$pr" | jq -r '.created_at')

    # Check if the PR author is in the list of specified usernames
    if [[ " ${USERNAMES[@]} " =~ " $pr_author " ]]; then
        # Append the PR details (including Created Date) into the CSV file
        echo "$pr_number,\"$pr_title\",$pr_author,$pr_url,$pr_created" >> "$output_file"
    fi
done

echo "Finished processing pull requests and saved to $output_file."
