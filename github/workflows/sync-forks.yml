name: Sync All Forks

on:
  workflow_dispatch: # Allows manual trigger with a button
  schedule:
    - cron: '0 0 * * 0' # Optional: runs weekly on Sunday at midnight UTC

jobs:
  sync-forks:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout a temporary repo
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository }}

      - name: Set up Git
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
      
      - name: Sync all forks
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Get all repositories for the authenticated user
          REPOS=$(gh repo list --json name,nameWithOwner,isInOrganization --limit 1000)
          
          echo "$REPOS" | jq -r '.[] | select(.isInOrganization == false) | .nameWithOwner' | while read repo; do
            echo "Processing: $repo"
            
            # Clone the repo
            gh repo clone "$repo" "/tmp/$repo" -- --quiet 2>/dev/null || continue
            
            cd "/tmp/$repo"
            
            # Get the upstream URL (assumes fork naming convention)
            UPSTREAM=$(gh repo view --json parent --jq '.parent.nameWithOwner' 2>/dev/null)
            
            if [ -z "$UPSTREAM" ]; then
              echo "Skipping $repo - not a fork or no upstream"
              cd - > /dev/null
              continue
            fi
            
            echo "Syncing $repo with upstream: $UPSTREAM"
            
            # Add upstream remote
            git remote add upstream "https://github.com/$UPSTREAM.git" 2>/dev/null || git remote set-url upstream "https://github.com/$UPSTREAM.git"
            
            # Fetch upstream
            git fetch upstream
            
            # Get the default branch
            DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
            
            # Checkout default branch
            git checkout "$DEFAULT_BRANCH" 2>/dev/null || git checkout -b "$DEFAULT_BRANCH" "origin/$DEFAULT_BRANCH"
            
            # Merge upstream changes
            if git merge "upstream/$DEFAULT_BRANCH" --no-edit; then
              # Push changes back to fork
              git push origin "$DEFAULT_BRANCH"
              echo "✓ Successfully synced $repo"
            else
              echo "✗ Merge conflict in $repo - manual review needed"
            fi
            
            cd - > /dev/null
          done