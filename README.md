#!/bin/bash

# GitHub repository details
REPO_OWNER="your-username-or-organization"
REPO_NAME="your-repo-name"
GITHUB_TOKEN="your-github-token"

# List of usernames to filter by (teammates)
USERNAMES=("user1" "user2" "user3")

# Output CSV file
output_file="pull_requests.csv"

# Write CSV header (include Created Date)
echo "PR Number,Title,Author,URL,Status,Created Date" > "$output_file"

# Fetch the last 100 PRs
echo "Fetching the last 100 pull requests..."
prs=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=100")

# Loop through each PR by extracting PR numbers and authors using jq for proper JSON parsing
echo "Processing pull requests..."
for pr_number in $(echo "$prs" | grep '"number":' | awk '{print $2}' | sed 's/,//'); do
    # Extract the PR author
    pr_author=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"login":' | awk -F '"' '{print $4}')

    # Check if the PR author is in the list of specified usernames
    if [[ " ${USERNAMES[@]} " =~ " $pr_author " ]]; then
        # Fetch reviews for the PR
        reviews=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
            "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls/$pr_number/reviews")

        # Initialize a variable to track approval status
        approved=false

        # Loop through the reviews and check for approval status
        for state in $(echo "$reviews" | grep '"state":' | awk -F '"' '{print $4}'); do
            if [[ "$state" == "APPROVED" ]]; then
                approved=true
                break
            fi
        done

        # Extract the PR title, URL, and created_at (created date) properly
        pr_title=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"title":' | awk -F '"' '{print $4}')
        pr_url=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"html_url":' | awk -F '"' '{print $4}')
        pr_created=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"created_at":' | awk -F '"' '{print $4}')

        # Determine approval status
        if [[ "$approved" == true ]]; then
            status="Approved"
        else
            status="Not Approved"
        fi

        # Write the PR details (including Created Date) into the CSV file
        echo "$pr_number,\"$pr_title\",$pr_author,$pr_url,$status,$pr_created" >> "$output_file"
    fi
done

echo "Finished processing pull requests and saved to $output_file."
