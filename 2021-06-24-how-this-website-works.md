# How This Website Works

I have created this website as a way for me to play around with different technologies. It is not
intended to be a masterpiece, especially on the UI design-end, but it has a few neat features you
may be interested to read about. The following sections list some aspects of the website and its
deployment.

## Frontend

[React.js](https://reactjs.org/) and [Next.js](https://nextjs.org/) are used as web-frameworks
(using TypeScript, of course). They may seem overkill for a website as simple as this one, but as it
also handles dymanic content they are a good fit. More of this in the Backend section.

The main points about the frontend I want to touch on is that it is intended to be minimal and light
weight, readable and responsive.

### Styling

A few media queries are enough to make sure that the site also works on mobile devices. All of the
styling is done using [SCSS](https://sass-lang.com/), which supports features like mixins to avoid
deduplication, for example for the media queries. CSS grid is used for the main layout and the
semantic HTML elements and of course flexbox for many centering use cases.

### Pages

Each page after the root page `/` is defined in Next.js's `pages` directory, which makes it serve
them as distinct pages while still using JavaScript loading and URL replacement instead of forcing a
slower full browser-reload on a page change.

```
pages
├── 404.tsx
├── about.tsx
├── api
│   └── health.ts
├── _app.tsx
├── index.tsx
├── portfolio
│   ├── index.tsx
│   └── legacy.tsx
└── posts
    ├── index.tsx
    └── [pid].tsx
```

### Email Obfuscation

On my [previous website](https://lolei.github.io) I had the problem that my email address was easily
parsed/scraped by spam bots, resulting in many spam emails. This time I wanted to avoid that and
obfuscate the address, but without impacting usability for real users. Methods such as inserting
`[at]` instead of `@` were therefore discarded. A good solution is provided by
[react-obfuscate](https://github.com/coston/react-obfuscate), which works by having the email
address value reversed in the HTML, and only when the `mailto` link is hovered or clicked, the value
is re-reversed via CSS. This should in theory prevent bots from getting the address. I will observe
the effectiveness of this solution, although it may prove difficult, since I'm sure that my address
is already in many spam databases.

```jsx
<div className={styles.contact}>
  <span>Email:</span>{" "}
  <Obfuscate
    email={props.email}
    headers={{
      subject: "Contact from Homepage",
    }}
  />
</div>
```

### Markdown Rendering

Since the posts are retrieved from an external source, a convenient markup language in which to
write the posts is Markdown. The Markdown could of course be converted to HTML with something like
pandoc, but since this is a React project the component
[react-markdown](https://github.com/remarkjs/react-markdown) is used for that purpose.

This is again a reason why React is useful even on smaller websites, as these utility components are
very easy to install and a hassle to replicate on your own.

```jsx
// components includes some more code in order to enable syntax highlighting
<ReactMarkdown plugins={[gfm]} components={components}>
  {props.postData.content}
</ReactMarkdown>
```

## Backend

Next.js also serves as the server/backend for the website. I've touched on Next.js's routing in the
previous section.

### Data Fetching

Another feature used is Next.js's data-fetching `getServerSideProps` method, which lets you define
code to retrieve data in a component in the `pages` directory, which can then be passed to that
component. This code is however executed on the server only and not included in the browser bundle.
It is also used to pre-fetch the data on actions like hovering over a link. This reduces the
time-to-interaction, as a page does not have to wait for all data to be loaded until it is displayed
and/or the elements of the page are filled in.

```ts
// Fetching the post list
export const getServerSideProps: GetServerSideProps = async () => {
  const database = Cache.Instance;

  const repopulateIfNecessary = async (): Promise<void> => {
    if (await database.datastorePostList.needsRepopulate()) {
      await database.datastorePostList.populate();
    }
  };

  const posts = await database.datastorePostList.getAll();
  repopulateIfNecessary();

  if (posts == null || posts.length === 0) {
    return {
      notFound: true,
    };
  }

  return {
    props: {
      postListings: posts.sort((a, b) => {
        return a.name > b.name ? -1 : 1;
      }),
    },
  };
};
```

### Dynamic Elements

The current dynamic elements are the posts and the items in the portfolio. The posts are retrieved
from a [repository](https://github.com/LoLei/posts) of Markdown files, the portfolio items from a
JSON document containing links to GitHub and GitLab repositories, from which the information like
the description, stars and the main language is fetched. This is done via the GitHub and GitLab API,
within the previously mentioned Next.js server-side data fetching methods. However, since those APIs
impose rate limits on their users the results of an API fetch is stored in an in-memory database,
and only updated once a certain period of time has passed and a new request is made to that section
of that website. This makes sure the API limits are never reached. Since this "changing" data is
retrieved from outside of the website, adding posts for example is as easy as committing a new
Markdown file to the posts repository.

## CI/CD

### GitHub Actions

[Github Actions](https://github.com/features/actions) is used to enable CI/CD. There are two
workflows, [one](https://github.com/LoLei/homepage/blob/main/.github/workflows/preliminary.yml)
which is run on every push to master and on pull requests to master from non-fork sources, which has
a few preliminary checks such as formatting, linting, building and running tests, and
[another](https://github.com/LoLei/homepage/blob/main/.github/workflows/build-publish-deploy.yml)
workflow, which runs when tags are pushed via
[bash2version](https://github.com/maxaudron/bash2version), which builds the application into a
container image, pushes it to the GitHub Container Registry, and deploys the new image to the
Kubernetes cluster, which is described in further detail in the next top-level section.

### Container

The [containerfile](https://github.com/LoLei/homepage/blob/main/Containerfile) is based on the one
officially recommended by Next.js's
[documentation](https://nextjs.org/docs/deployment#docker-image). It uses a multi-stage build in
order to make the image as small as possible. In addition to that, since when building the image in
a fresh virtual environment within a GitHub Actions workflow layer caching cannot be conventionally
used, the [build-push-action](https://github.com/docker/build-push-action) is utilized, which uses
buildx/BuildKit to enable caching again.

The ability to easily include community-made "actions" in your workflow is one very convenient
feature of GitHub Actions, and a couple of other ones are used as well.

The built image is automatically pushed to the [GitHub Container
Registry](https://github.com/LoLei/homepage/pkgs/container/homepage) as a package, from which it can
be pulled by Kubernetes, or other users.

## Deployment

The main Kubernetes [deployment manifest](https://github.com/LoLei/homepage/blob/main/k8s/deployment.yaml)
consists of a fairly standard Deployment, Service and Ingress setup. The more noteworthy parts out
pointed out below.

### Secrets

One interesting aspect is the usage of [sealed secrets](https://github.com/bitnami-labs/sealed-secrets).
Since I wanted a way to also track my Kubernetes secrets with git, but storing them publicly would
be insecure due to plainly visible base64-encoded secret values, I found sealed-secrets which
provides a solution that encrypts the secret values. These encrypted secrets can be pushed to a
public repository. In the cluster they can be used as "recipes" to create real secrets, since the
cluster-localized sealed-secrets controller has the key to decrypt and "unseal" the values.

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: homepage-secret
  namespace: default
spec:
  encryptedData:
    GITHUB_PAT: AgAD3k7z3I3QimLVBujPitFm9bLzot6B8vCoYE1lxxjSMPFW/v...
  template:
    data: null
    metadata:
      annotations:
        sealedsecrets.bitnami.com/managed: "true"
      creationTimestamp: null
      name: homepage-secret
      namespace: default
    type: Opaque
```

### SSL Certificates

Let's Encrypt's SSL certificates are handled by [cert-manager](https://cert-manager.io/). Provided a
[cluster issuer](https://github.com/LoLei/k8s/blob/master/do/production_issuer.yaml), it creates a
certificate and secret for each Ingress resource that specifies `cert-manager.io/cluster-issuer: <issuer_name>` in its annotations.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homepage-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - lolei.dev
      secretName: homepage-tls # This secret will be created automatically
  rules:
    - host: lolei.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homepage-svc
                port:
                  number: 80
```

# Conclusion

I hope this gives some insight into the inner-workings of this website and why I chose to build it.
In the future it may be used to host more blog posts, however it may also be taken down temporarily
in order to save hosting costs, because realistically, not many people are constantly looking at my
content. At least not at the moment 😉.
