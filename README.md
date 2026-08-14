# blog.orn-dijkstra.no

Personal Hugo blog deployed by GitHub Pages.

## Write

```sh
hugo new content work/my-post.md
hugo server -D
```

Use `personal/` instead of `work/` for personal posts. Set `draft: false` when a post is ready.

## Publish

1. Create a GitHub repository and push this project to its `main` branch.
2. In **Settings → Pages**, set **Source** to **GitHub Actions**.
3. At your DNS provider, add this record:

| Type | Name | Value |
| --- | --- | --- |
| CNAME | `blog` | `<github-username>.github.io` |

4. In **Settings → Pages**, confirm `blog.orn-dijkstra.no` as the custom domain and enable **Enforce HTTPS** after DNS has propagated.
