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

# Initial page number
page=5
per_page=100
next_page_url="https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/pulls?state=all&per_page=$per_page&page=$page"

# Function to fetch and process PRs from a given URL
fetch_prs() {
    local url=$1
    prs=$(curl -s -I -H "Authorization: token $GITHUB_TOKEN" "$url")

    # Extract the body (PRs data) and the Link header (for pagination)
    prs_body=$(curl -s -H "Authorization: token $GITHUB_TOKEN" "$url")
    prs_link_header=$(echo "$prs" | grep "Link:")

    # Loop through each PR
    for pr_number in $(echo "$prs_body" | grep '"number":' | awk '{print $2}' | sed 's/,//'); do
        pr_author=$(echo "$prs_body" | grep -A 10 "\"number\": $pr_number" | grep '"login":' | awk -F '"' '{print $4}')
        
        # Check if the PR author is in the list of specified usernames
        if [[ " ${USERNAMES[@]} " =~ " $pr_author " ]]; then
            pr_title=$(echo "$prs_body" | grep -A 10 "\"number\": $pr_number" | grep '"title":' | awk -F '"' '{print $4}')
            pr_url=$(echo "$prs_body" | grep -A 10 "\"number\": $pr_number" | grep '"html_url":' | awk -F '"' '{print $4}')
            pr_created=$(echo "$prs_body" | grep -A 10 "\"number\": $pr_number" | grep '"created_at":' | awk -F '"' '{print $4}')
            
            # Append the PR details into the CSV file
            echo "$pr_number,\"$pr_title\",$pr_author,$pr_url,$pr_created" >> "$output_file"
        fi
    done

    # Return the Link header (for pagination)
    echo "$prs_link_header"
}

# Loop through all pages using the Link header
while [ -n "$next_page_url" ]; do
    echo "Fetching PRs from: $next_page_url"
    
    # Fetch the PRs and get the Link header for the next page
    link_header=$(fetch_prs "$next_page_url")

    # Extract the next page URL from the Link header
    next_page_url=$(echo "$link_header" | grep -o '<[^>]*>; rel="next"' | sed -e 's/^<//' -e 's/>; rel="next"//')

    # If there's no next page, we stop
    if [ -z "$next_page_url" ]; then
        echo "All pages processed."
        break
    fi
done

echo "Finished processing all pull requests and saved to $output_file."
