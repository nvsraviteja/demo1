#!/bin/bash

# GitHub repository details
REPO_OWNER="your-username-or-organization"
REPO_NAME="your-repo-name"
GITHUB_TOKEN="your-github-token"

# List of usernames to filter by (teammates)
USERNAMES=("user1" "user2" "user3")

# Fetch the last 100 PRs
echo "Fetching the last 100 pull requests..."
prs=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=100")

# Loop through each PR by extracting PR numbers and authors using grep and awk
echo "Processing pull requests..."
for pr_number in $(echo "$prs" | grep '"number":' | awk '{print $2}' | sed 's/,//'); do
    # Extract the PR author
    pr_author=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"user":' -A 1 | grep '"login":' | awk -F '"' '{print $4}')

    # Check if the PR author is in the list of specified usernames
    if [[ " ${USERNAMES[@]} " =~ " $pr_author " ]]; then
        # Fetch reviews for the PR
        reviews=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
            "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls/$pr_number/reviews")

        # Check if the PR has any reviews
        if echo "$reviews" | grep -q '"id":'; then
            # Extract the PR title and URL
            pr_title=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"title":' | awk -F '"' '{print $4}')
            pr_url=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"html_url":' | awk -F '"' '{print $4}')

            # Print the PR details if it matches the usernames
            echo "PR #$pr_number: $pr_title"
            echo "Author: $pr_author"
            echo "URL: $pr_url"
            echo ""
        fi
    fi
done

echo "Finished processing filtered pull requests."
