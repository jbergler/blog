+++
date = "2018-09-05T02:58:16Z"
title = "git-new-branch"
+++

When I'm coding I regularly find myself switching between different tasks.
Somewhere along the way I'd picked up a habit of creating branches with non descriptive names such as `list-bug`, `tests`, `bugfix` and so on.
Since most of my branches were short lived this was mostly fine, but over time I found myself tinkering with various experiments more frequently. Slowly the number of branches in my checkouts grew and it became harder and harder to find something I was playing around with a few weeks ago. 

My solution was to write a quick shell function to create new branches with a standard naming scheme, so I present: `gnb`

```bash
% which gnb
gnb () {
	branch="wip/$(date +%Y-%m-%d)/${1:-untitled-$(date +%H%M)}" 
	upstream="${2:-origin/master}" 
	git fetch $upstream
	git checkout --track -b $branch $upstream
}
```

The end result: I now have hundreds of branches, neatly ordered by time.

If you like it, steal it. And remember, the right solution is the one that makes you more productive, so take the time to make your environment work for you.
