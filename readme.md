# Theoretical foundations for server-side rendering and static-rendering
Eric Burel, LBKE, https://www.lbke.fr
Draft paper

## Abstract

This paper contributions are threefold. 
First, we propose formal definitions for various web content rendering approaches that powers the "jamstack": prerendering, server-side rendering, static rendering... Many definitions exist in the industry but none is canonical and some are even contradictory. Beyond wording details and framework specificities, we strive to unify those concepts within a common, broader understanding.
Then, we model server-side rendering mathematically as a binding between two sets: the set of all possible requests, and the set of responses from the server. From this model, we derive rules that define for which requests pre-rendering is possible or not.
Finally, we develop an API that should encompasses all possible server-rendering approaches, should they happen at build-time, request-time, or somewhere in-between. We demonstrate a partial implementation of this API using Next.js 12.
From this study, we conclude that the division between static and request-time rendering is mostly fictional and sums up to a problem of cache configuration, and that build-time rendering in particular has been vastly underused until recently.

## Related work

If you believe a paper, article or resource should appear in this section, please reach us out: contact@lbke.fr.
We use the definitions provided by the 2 current big players of the Jamstack ecosystem as our work-basis, namely Netlify, through the Jamstack.org website (https://jamstack.org/) and Vercel, through the Next.js framework (https://nextjs.org/).
At the time of writing, Next.js proposes the most complete set of features for server-side rendering, and therefore has been chosen as the basis of our implementation.

A good example of concept that requires clarification is "prerendering". For Vercel, "prerendering" means server-side rendering, whether it happens at build-time or request-time or somewhere in between. For Jamstack.org, "prerendering" only includes build-time rendering aka "static rendering". See https://github.com/jamstack/jamstack.org/issues/644

## Definitions

### Rendering
Rendering is the act of transforming a piece of content into either HTML or a DOM representation that can be displayed by a web browser, client-side.

### Server-side rendering

There are many definitions of "server-side rendering" (SSR) or in the wild. Let's stick to the most basic one : server-side rendering is rendering a web page, on a server, as opposed to rendering in the client browser.
Server-side rendering is as ancient as the web itself, and was almost the only way of rendering HTML content programmatically until client-side JavaScript got serious with the introduction of Chrome V8 engine in 2008.

This includes:
- build-time server rendering, also known as static rendering, or static site generation (SSG). This is when you render the pages of your website when you publish it, once for all.
- request-time server rendering, also known as just server-side rendering (SSR). This is when you render the page every time someone request it.
- no rendering at all, when content is stored directly as HTML or text. It's an edge case yet falls into the server-side rendering category, as it doesn't involve any kind of client-side rendering.

### Render moment and the not-so-special case of static rendering

Rendering the content at build-time, or not rendering it at all, is considered as a special case. Therefore, in the industry, SSR is often synonymous to request-time server-rendering, while the terme "static" is preferred for build-time rendering or no rendering at all.

From the server owner perspective, this distinction makes sense: the cost structure of per-request rendering and build-time rendering are very different. Per-request has no upfront cost, but a needs a render for each new request ; build-time has an upfront cost, but do not need additional computation afterward.

However, from the web developer perspective, this distinction doesn't really stand. The only difference is when the render happens. But in both scenarios, this render happens server-side.

In this paper, we'll stick to the web developer standpoint. Therefore, to us, server-rendering includes per-request server-rendering (aka "SSR") and build-time rendering (aka "Static Site Generation (SSG)", "static rendering"...).

Let's introduce the concept of "render moment": the time when the render happens, rather than the place.
 
We advocate for a lighter distinction between patterns whose only difference lies in the render moment, such as per-request SSR versus build-time SSR. 
The "render moment" should be treated as a matter of configuration, and not as the fundamental cornerstone of the "Jamstack" philosophy.

The remainder of this paper is focused on server-side rendering only.

### Technical definitions
In this section, we'll dive into the deeper technical details of server-side rendering needed for implementation.

#### Page, template and props

Let's call the result of render is a page. When the render happens server-side, the page is a combination of HTML, JavaScript and CSS code.

A template is a generic web page, that expects some values to generate actual HTML and styles. Those values can be called "props". 
The template could be typically a React component, or a template written in more classical language, like EJS, PUG, PHP...

So, a page is a rendered template. Like a text with blanks, whose blanks have been filled.

#### Request as input of a server-render, be it build-time or request-time

A request is the basic input of request-time server-rendering. It's more precisely an HTTP request, most of the time triggered by a human user through a web browser. 

A request can be seen as a set of various attributes. 

However, there's a fact often overlooked in the Jamstack ecosystem: in accordance to our definition of server-side rendering, build-time server rendering *also* expects a request as its input. Not only because the user has to initially query a server, via an URL, to get some content, but also because this URL is actually used by the server to retrieve the right piece of content.


For instance, imagine a blog with 10 articles, that is statically built using a React framework such as Gatsby or Next.js Each article has its own URL. So when the user types "your-super-blog.whatever/article-1" they get article 1.
The URL is one of the attribute of the user request, and this attribute helps the server redirect the user to the right page but also the build engine to pre-render the right article for each page.

The main difference with traditional per-request server-side rendering, is that the build-time rendering approach usually only uses the URL, and ignore the rest of the HTTP request. We argue that even though existing frameworks systematically waste the remainder of the HTTP request, this is not a fatality. Cookies, headers and other request attributes could also be used to select the right piece of content.

We should keep in mind that there is no such thing as a "serverless" website. Static websites are relying on very simple servers, that just do some redirections, but there are always servers and HTTP requests around the corner.

Therefore, build-time rendering can be redefined as a server-side pre-computation of a handful requests the end-user may or may not make against the server. The initial input is still an HTTP request, only the render moment differs with per-request SSR.

#### Steps of server-side rendering at runtime

When a user access a web page, per-request server-side rendering can be split in following steps:

1. For a given request, select the right template depending on the HTTP request attributes. 
2. Compute the props of the template based on user's request. This step simply translates the HTTP request into a set of attributes that makes more sense in the business domain. Technically, this is not strictly needed for server-side rendering, the request could be directly passed to the render function.
3. Render the page, by feeding the page template with the right props.

Request-time rendering does this for every HTTP request.

Build-time rendering does mainly the same thing, but in advance, at build-time, for a handful of precomputed HTTP requests. Then, when an HTTP request is received, the server will simply return the right rendered page. 

Build-time rendering behaves as follow:

0. Prerender the pages for various requests (see steps above)
1. Same as for per-request SSR: select the right template based on the request. Most often based on the URL, but it could be based on other request attributes such as cookies, request headers, etc.
2.  Return the pre-rendered content (no need for an additional computation)

#### Formally

A $renderer$ function takes requests as input and returns pages.

$$
renderer(req) \mapsto page\\
\left\{
        \begin{array}{ll}
req = (attr_i)\\
page = (HTML, CSS, JS)
    \end{array}
\right.
$$

Request-time rendering does this for every request. Build-time rendering caches the page once for all during the build.

If we go step by step, we can also define the following intermediate function:
$$
templateGetter(req) \mapsto template\\
propsGetter(req)\mapsto props\\
template(props) \mapsto page\\
renderer(req) = templateGetter(req)(propsGetter(req))\\
\left\{
        \begin{array}{ll}
req = (attr_i)\\
props = (prop_j)\\
page = (HTML, CSS, JS)
    \end{array}
\right.
$$

We do not really care about the final rendered value here. That's a matter of implementation. The template choice is also most often directly related to the URL. Therefore, the $propsGetter$ is the most important function here, the function that computes the values needed for rendering based on the request.

Attributes of a request may be of different nature. We can represent them as a flat dictionary or a vector.

- Discrete and finite : the user id, some url parameters
- Infinite, or continuous : the current date, some other url parameters

So, there is an infinite number of requests if you consider all parameters, but you can build finite subsets if you consider only certain attributes of the request.

#### Takeaways

- server-side rendering is just normal request processing. We don't really care about the technology, any app with server-rendering is just a function that processes requests and outputs some results. If the result happens to be some HTML, CSS and JS, we call that rendering, but there is no strong difference with any other kind of API. 
- what matters are the value you will use to fill the blanks your template: the props.
- build-time rendering, or "static" rendering, is just precomputed server-side rendering
- the difference between build-time rendering ("static") and request-time rendering ("ssr") is mostly based on the nature of the request attributes you'll want to consider to compute the result.

## When build-time rendering is possible

### Build-time rendering is precomputed request-time rendering

At build time, there is no HTTP request happening. Yet, build-time rendering is still heavily based on the concept of request, as explained before: it's just precomputation of a bunch of requests.

If you build 10 pages for 10 articles, you are precomputing 10 requests, one for each URL. Existing frameworks are often adopting this "URL" vision of build-time rendering. 

Yet, they could also consider other request attributes such as cookies, that would work exactly the same. For instance, you could prerender a dark mode and light mode version of your interface, based on a cookie.

Therefore, to keep our definition consistent, we can suppose that the render function is still using a request as input, except that it is limited to parameters you can know at build-time.

### What can be built: the 3 rules of build-eligibility

Intuitively, build-eligibility depends a lot on the attributes you consider in the request. 
If you have 2 modes, light and dark, that works. 10 articles on your blog, that works. 
But if you want to prerender one page per atom in the universe, you'll be in trouble.

Let's try to figure when build-time rendering is possible or not more formally. Since build-time rendering is precomputing some renders for a set of requests, let's define the "build-eligibility" in terms of ensemble of possible requests instead. 

To be eligible for build-time rendering, our set of requests must have following properties:

- It's a subset of the set of all possible requests (the requests are valids and make sense, like URL are correct URLs etc.)
- It must be a finite set, otherwise build time would be infinite
- Values should be known at build time and stay constant afterward


### Formally

Let's note $R$ the set of all possible HTTP requests in the world.

Let's note $R_{getter}$ the set of all requests that a $propsGetter$ takes as input. It depends on which part of the request $propsGetter$ is actually using to compute the props.

For instance, if we only use attribute 1 (say, the URL), and attribute 4 (say, the cookie that sets light or dark mode):
$$
R_{getter} = R_{\{attr_1, attr_4\}} \iff \forall req \in R_{getter}; propsGetter(req) = propsGetter({\{attr_1, attr_4\}})\\
$$

Let's note $RB$ the set of all subsets of $R$ that are build-eligible. 
Picture it as any valid combination of attributes that can be used to compute the props, and still respect the 3 rules of build-eligibility. This is not useful to our reasonning but facilitates notation a lot, build-eligiblity can be written like this: $R_{getter} \subset RB$.

So, a valid build-time request is any set of attributes that belongs to any valid $R_{build}$ ensemble included in $RB$. 

A build-eligible set of requests would have to respect those 3 conditions:

$$
R_{build} \subset R\\
|R_{build}| < \infty\\
R_{build}(t) = R_{build}
$$

Rule 1 is that obviously the request should make sense and be a "possible" request. 

Rule 2 is the "rule of finiteness"

Rule 3 is the "rule of staticness".

If $R_{getter}$ respects all 3 conditions, it's a build-eligible renderer, and it belongs to $RB$. 

Build-time rendering is applying $renderer$ (and thus $propsGetter$) to all $req$ included in $R_{getter}$, supposing that $R_{getter} \subset RB$.

Congratulations, you can enjoy static rendering from an ensemblist point of view.

## Implementation

Here are the typings and the final build-time rendering function:

```ts
type Req = Object
type Props = Object
type Page = string // some HTML

type RBuildComputer = () => Array<Req> // must respect the build-eligbility conditions
type Template = (Props) => Page 
type TemplateGetter = (req: Req) => Template
type PropsGetter = (req: Req) => Props

type Renderer = (req: Req) => Page // = TemplateGetter(Req)(PropsGetter(Req))
```

Example:
```ts
function rBuildComputer () { return [{url: "article/1"}, { url: "article/2"}] }

function articleTemplate ({articleId}) { return  `Reading article ${articleId}` }
function templateGetter (req) {
    if (req.match(/article/)) return articleTemplate
    return () => `Default template` 
}
function propsGetter (req) {
    return { articleId: req.params.articleId }
}
function renderer = (req){
    return templateGetter(req)(propsGetter(req))
} 
```

The `rBuildComputer` function doesn't feel very natural. That's because requests are supposed to be random events, triggered by the user, so we are not used to list all the possible requests for a given endpoint.

Also, when actually serving the pages, this means you still need some request processing logic. The final HTML/CSS/JS result may be cached, the result of the `renderer` function, but you still need to check the request URL everytime to get the right page in this example. 

We'll describe how Next.js solves a simplified version of this problem later-on.

### Takeaways

- By selecting the attributes you consider as input of your renderer function, you define an $R_{getter}$ set. If it respects the 3 build rules, you can enjoy static rendering.
- Example: rendering a finite number of blog articles. $R_{getter}$ is the list of all your articles URL, it's finite, doesn't change. If you write a new article, you can of course rebuild to get fresh data (we'll dig that problem later).
- This is theoretical, in real life computing the set of all possible requests feels unnatural.
- There is no such thing as a website without a server. Static hosts are just extremely basic server that can only process the URL part of the request, but they still do process the request. 

## Rendering at request time

As soon as the request attributes you use to compute violates one of the 3 conditions to be a build-eligible set... you cannot build-time render anymore :( You are good to do some request-time server-side rendering, meaning you'll need to compute the render function for each incoming request.

Examples :

- One attribute is infinite. Say, your page is displaying the current date. You cannot prebuild for all dates.
- One attribute is time-dependent. Request time is the obvious example here, but it also violates the finiteness property so it's not that interesting. A better example would be any kind of dynamic attribute. For example, the user id. The list of users evolves each time someone sign up on your website, so you cannot rebuild every time someone signs up.

Let's focus on this "you cannot rebuild every time". Actually, this definition is a bit trickier.

### The 4th dynamic rule of build-eligibility

When you add a new article for your awesome blog, or someone signs up to your equally awesome SaaS product, you are making your build-time rendered website obsolete. You need a rebuild.

Suppose that rendering time is constant for all requests, for the sake of simplicity. 

That rendering time must be smaller than the time between 2 changes of your list of articles, otherwise you will not have enough time to rebuild your website.

Examples:

- the list of articles on your personal blog only evolves daily/weekly. A rebuild takes a few minutes => articles are eligible for build-time rendering
- the list of users on your brilliant SaaS product evolves a lot, you can get 10 new users a day (or more, who knows) => you are not eligible

### Formally

Let's call it $t_{render}$ the execution time of $renderer$.

Let's note the time for a variation of $R_{getter}$ to happen $t_{\Delta R_{getter}}$.

$$
t_{render} > t_{\Delta R_{getter}} \implies R_{getter} \not\subset RB  \\
$$

It means that if you can rebuild faster than the list of possible values, for all attributes, the build-eligibility still holds. But if you rebuild slower, it doesn't not, you need request-time rendering.

### Takeaways

- there are actually 4 rules for build-eligibility, one of them depends on how fast the possible values evolves for each input attributes. Build-time rendering makes sense only for slowly changing values (a few seconds being a minimum, a few minutes more appropriate, a few days the best).
- request-time server rendering is more an "exception" than the default. Build-time rendering should be preferred whenever possible.

## Next.js vision of rendering

Let's apply our model to Next.js.

### A page = a template + a props getter function

In Next.js, a "page" is a React component tied to props computing functions, either build time or request time.

So, in our terminology, a Next.js "page" is a template + a props getter function.

### Static Site Generation

Static Site Generation is a form build-time rendering. However, it does accept only one attribute as input: the request URL. 

This includes the URL parameters as well. So, a page always have a base path, for instance, `/blog/articles/:id `. In Next.js, this corresponds specifically to its position in the `pages` folder. But it can also have "parameterized paths", which are called dynamic routes : that's just the base path + some route parameters. For instance, `/blog/articles/12`.

So, Next implicitly defines an additional build-time eligibility rule like follow:
$$
R_{getter} \subset RB \implies \forall req \in R_{getter}; propsGetter(req)= propsGetter(req_{\{url\}})
$$
The props must only depends on the considered route or URL. Otherwise you can't use the static props computation functions provided by Next.

If we consider that all URLs for a page are known at build-time, we can render the page once for each possible URL. At runtime, all subsequent request on the same URL will get the same page.

#### Implementation

The big advantage over our generic version? This is easier to understand for Next.js end users. The request processing is limited to a bare minimum, since you only need to look at the base path to find the template, and the route parameters to compute the props. 

Intuitively, it's easy to precompute all the possible paths for an app. Even if the values are dynamic, you simply need a few requests, for instance to get the list of articles from your CRM.

The limitation is however that in order to get different props, you need a different URL. So if you want to prerender a light and dark mode for the same route, you would need an ugly route parameter just for that. You cannot get different props based on a cookie or a query parameter. 

Also, a lot of valid build-time attributes cannot be obtained just by looking at the URL. You would also need to process the request cookies for instance, to tell if the user is authenticated or not.

### Server-side rendering

Nothing fancy here, it's just classical SSR. Take a deep breath, the last part is the interesting one.

### Incremental Static Regeneration

Formally, Incremental Static Rendering alleviates both the rule of finiteness and the rule of staticness for build-eligibility. Instead of rendering all the pages at build-time, you render them only on-demand. You can also rerender the pages more frequently, without rebuilding everything.

Since the number of requests that can hit your website is finite in a finite amount of time, it allows you to stretch the build time infinitely. 

Note: there is still a very minor hypothesis, that people don't spam your website with so many different requests that it saturates your ability to render the pages. More formally, that they don't map $R_{getter}$ faster than $|R_{getter}|*t_{render}$. 

And also, that the possible values still evolves relatively slowly, otherwise user will still get stale data. If the cache time to live is too short, you end up with usual server-side rendering, which defeats the purpose of ISR.

There is still a strong limitation: props are still entirely defined by the URL. You cannot process the request with a custom function in the current implementation.

## A unified API for server-rendering

Based on this model, here is what could be a unified view of SSR and SSG, for a Next.js page:
```ts
// The actual page, written with a SPA framework like React, Vue, Svelte...
type Page = (props: Props) => [HTML, CSS, JS]

// Compute the requests we want to prerender
// Example : all the articles of a blog
type computePossibleRequests = () => Promise<Array<Request>>

// For a given request, compute the right props
// Example : get the article 42 when URL is « /article/42 »
type propsGetter = (req: Request)=> Promise<Props>

// Time the rendered page should stay in the cache for a given set of props
type TTL : Number
```
Yes, build-time static rendering is just server-side rendering with a cache + precomputed requests. Tada. You don't need really have to differentiate `getServerSideProps` and `getStaticProps`. 

And an exemple implementation for paid pages on a blog:
```ts
export const BlogPage = (props: Props) => (
<div>
	<p>{props.paidArticle.title}</p>
	<p>{props.paidArticle.content}</p>
</div>
)

// The requests you'd like to precompute
// In Next.js, this is currently limited to a list of URLs for public content.
// Here, we also want to pre-render private paid articles.
// When the user opens the page:
// 1. An upfront middleware/server checks authentication, subscription
// and set the right headers securely (X-PAID in this example).
// 2. If the url params + headers are matching an existing pre-rendered
// request, the page will be rendered immediately.
// 2bis. If there is no match, it will lead to a 404 (but you could
// have a smarter strategy, like letting the upfront server redirect the user
// to a "/subscription" page)
export async function computePossibleRequests = (): Array<Request> => {
    const paidArticles = await fetchPaidArticles()
    return paidArticles.map((article => ({
       urlParams: { id: article.id},
       // those articles are only available to paid users
       header: {"X-PAID": true}
    })
}

// This function will be run during static render 
// for all the private articles of the database, 
// and also be run during request-time render
// There is no way to tell whether it's static or request-time render, 
// because you don't need to! 
// Think of Next.js getInitialProps, but with the request also available
// during static render
export async function propsGetter(req: Request): Props {
    const { urlParams, header } = req
    const privateArticle = await fetchPrivateArticle(urlParams.id)
    return { privateArticle }
}
// we rerun propsGetter every minute to get a fresh version of the article
export const TTL_MS = 60000
```
Possible scenarios depending on the caching strategy:
- $TTL = \infty$ => this is static rendering. You must define `computePossibleRequests` as well, to get the list of pages to render.
- $TTL = 0$ => this is server-side rendering.
- $TTL = X; 0 < X < \infty$ => this is incremental static regeneration. You may want to prerender some pages as well.
- If `propsGetter` always return a new value (say it includes current time for instance), TTL should be set at zero. Otherwise memory will explode because of useless caching.
- You can always define `computePossibleRequests` to precompute some pages at build-time, for an hybridation between static render and server render (that's the point of ISR).

## Implementation

No existing framework implements the API we propose. However, the introduction of a middleware system in Next.js, coupled with "Incremental Static Regeneration", let's us get as close as possible from an optimal static rendering.
We use a palliative approach based on route parameters to simulate the precomputation of a set of request. The parameter encodes other attributes such as headers, cookies.
This implementation is further described in this informal article: https://blog.vulcanjs.org/render-anything-statically-with-next-js-and-the-megaparam-4039e66ffde

### Changelog
- 07/2021 - first draft
- 09/2021 - better example for the generic SSR API
- 11/2021 - Adding abstract, started to add related work, linking a working implementation
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTY0MTE4OTM3LC0yODQ1MzkxNDgsLTM1OD
MzOTgzLC0xNDU3ODYwMDQxLDEzMTYwOTYzMjMsLTQ1MzYwOTM4
NywtMTU2MzI2NjY2NCwxNjAyNzM5MzQ2LC0xMjYyMTYyMzM5LD
k5OTQ4MTg5MSwxOTMzMDUzNTMyLC0xNzg0MzUwMTk4XX0=
-->