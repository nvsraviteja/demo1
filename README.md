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

# Fetch the first page to determine total number of PRs
per_page=100
initial_response=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
    "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=$per_page&page=1")

# Extract the total number of PRs from the response headers (if available)
total_prs=$(echo "$initial_response" | grep '"number":' | wc -l)

# Calculate the number of pages based on total PRs
total_pages=$(( (total_prs + per_page - 1) / per_page )) # Ceiling division

# Function to fetch and process PRs from each page
fetch_prs() {
    local page=$1
    local prs=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
        "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=$per_page&page=$page")

    # Loop through each PR
    for pr_number in $(echo "$prs" | grep '"number":' | awk '{print $2}' | sed 's/,//'); do
        pr_author=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"login":' | awk -F '"' '{print $4}')
        
        # Check if the PR author is in the list of specified usernames
        if [[ " ${USERNAMES[@]} " =~ " $pr_author " ]]; then
            pr_title=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"title":' | awk -F '"' '{print $4}')
            pr_url=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"html_url":' | awk -F '"' '{print $4}')
            pr_created=$(echo "$prs" | grep -A 10 "\"number\": $pr_number" | grep '"created_at":' | awk -F '"' '{print $4}')
            
            # Append the PR details into the CSV file
            echo "$pr_number,\"$pr_title\",$pr_author,$pr_url,$pr_created" >> "$output_file"
        fi
    done
}

# Loop through all pages and fetch PRs
for page in $(seq 1 $total_pages); do
    echo "Fetching page $page of $total_pages..."
    fetch_prs $page
done

echo "Finished processing all pull requests and saved to $output_file."
