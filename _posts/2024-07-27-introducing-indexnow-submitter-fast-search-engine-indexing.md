---
layout: post
title: "Introducing IndexNow Submitter: Fast Search Engine Indexing"
date: 2024-07-27 11:00
category: Production
tags: ["nextjs", "indexnow", "fast", "index", "bing", "yandex", "naver", "seznam.cz", "yep"]
description: Introducing IndexNow Submitter, an npm module I created to boost your node app's SEO with faster indexing using IndexNow protocol.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Why IndexNow Submitter?](#why-indexnow-submitter)
- [Key Features](#key-features)
- [Getting Started](#getting-started)
   * [Basic Usage](#basic-usage)
   * [Submitting Multiple URLs](#submitting-multiple-urls)
   * [Parsing and Submitting from a Sitemap](#parsing-and-submitting-from-a-sitemap)
- [Command-Line Interface](#command-line-interface)
- [Advanced Features](#advanced-features)
   * [Caching](#caching)
   * [Analytics](#analytics)
- [Use Cases](#use-cases)
- [Conclusion](#conclusion)

<!-- TOC end -->

In the fast-paced world of web development and SEO, keeping search engines up-to-date with your latest content is crucial. That's why I'm excited to introduce [IndexNow Submitter](https://www.npmjs.com/package/indexnow-submitter), a powerful npm module designed to simplify and automate the process of submitting URLs to search engines using the [IndexNow](https://www.indexnow.org/) protocol.

<!-- TOC --><a href="#" name="why-indexnow-submitter"></a>
## Why IndexNow Submitter?

Search engine optimization is an ongoing battle. You work hard to create great content, but if search engines don't know about it, your efforts might go unnoticed. The IndexNow protocol aims to solve this problem by allowing websites to instantly notify search engines about new or updated content. However, implementing this protocol manually can be time-consuming and error-prone.

That's where IndexNow Submitter comes in. This npm module provides a simple, efficient, and flexible way to submit your URLs to search engines, ensuring that your content gets indexed as quickly as possible.

> The IndexNow protocol is implemented by several major search engines like bing, yandex, naver, seznam.cz and yz. However, google has not implement it yet.
{:.prompt-info}

<!-- TOC --><a href="#" name="key-features"></a>
## Key Features

- **Single and Batch URL Submission**: Submit individual URLs or batches of URLs with ease.
- **Sitemap Parsing**: Automatically extract and submit URLs from your XML sitemaps.
- **Caching**: Avoid redundant submissions with built-in caching.
- **Analytics**: Track your submission statistics to optimize your indexing strategy.
- **Rate Limiting**: Comply with search engine guidelines by controlling submission rates.
- **CLI Support**: Use as a module in your Node.js projects or as a command-line tool.
- **TypeScript Support**: Enjoy full TypeScript support for a better development experience.

<!-- TOC --><a href="#" name="getting-started"></a>
## Getting Started

To start using IndexNow Submitter, you can install it via npm:

```bash
npm install indexnow-submitter
```

<!-- TOC --><a href="#" name="basic-usage"></a>
### Basic Usage

Here's a simple example of how to use IndexNow Submitter in your Node.js project:

```typescript
import { IndexNowSubmitter } from 'indexnow-submitter';

const submitter = new IndexNowSubmitter({
  key: 'your-indexnow-api-key',
  host: 'your-website.com'
});

// Submit a single URL
submitter.submitSingleUrl('https://your-website.com/new-page')
  .then(() => console.log('URL submitted successfully'))
  .catch(error => console.error('Error submitting URL:', error));
```

<!-- TOC --><a href="#" name="submitting-multiple-urls"></a>
### Submitting Multiple URLs

Need to submit multiple URLs at once? No problem:

```typescript
const urls = [
  'https://your-website.com/page1',
  'https://your-website.com/page2',
  'https://your-website.com/page3'
];

submitter.submitUrls(urls)
  .then(() => console.log('URLs submitted successfully'))
  .catch(error => console.error('Error submitting URLs:', error));
```

<!-- TOC --><a href="#" name="parsing-and-submitting-from-a-sitemap"></a>
### Parsing and Submitting from a Sitemap

If you have a sitemap, IndexNow Submitter can parse it and submit all the URLs for you:

```typescript
submitter.submitFromSitemap('https://your-website.com/sitemap.xml')
  .then(() => console.log('Sitemap URLs submitted successfully'))
  .catch(error => console.error('Error submitting sitemap URLs:', error));
```

You can even specify a date to only submit URLs that have been modified since then:

```typescript
const modifiedSince = new Date('2023-01-01');
submitter.submitFromSitemap('https://your-website.com/sitemap.xml', modifiedSince)
  .then(() => console.log('Recent sitemap URLs submitted successfully'))
  .catch(error => console.error('Error submitting recent sitemap URLs:', error));
```

<!-- TOC --><a href="#" name="command-line-interface"></a>
## Command-Line Interface

IndexNow Submitter also comes with a CLI, making it easy to use in your scripts or CI/CD pipelines:

```bash
# Submit a single URL
INDEXNOW_KEY=your-api-key INDEXNOW_HOST=your-website.com npx indexnow-submitter submit https://your-website.com/new-page

# Submit URLs from a file
INDEXNOW_KEY=your-api-key INDEXNOW_HOST=your-website.com npx indexnow-submitter submit-file urls.txt

# Submit URLs from a sitemap
INDEXNOW_KEY=your-api-key INDEXNOW_HOST=your-website.com npx indexnow-submitter submit-sitemap https://your-website.com/sitemap.xml
```

<!-- TOC --><a href="#" name="advanced-features"></a>
## Advanced Features

<!-- TOC --><a href="#" name="caching"></a>
### Caching

IndexNow Submitter includes an intelligent caching system to prevent redundant submissions. This is particularly useful when you're running regular indexing jobs:

```typescript
const submitter = new IndexNowSubmitter({
  key: 'your-api-key',
  host: 'your-website.com',
  cacheTTL: 3600 // Cache URLs for 1 hour
});

// This URL will be submitted
await submitter.submitSingleUrl('https://your-website.com/page1');

// This won't be submitted again within the next hour
await submitter.submitSingleUrl('https://your-website.com/page1');
```

<!-- TOC --><a href="#" name="analytics"></a>
### Analytics

Want to know how your submissions are performing? IndexNow Submitter provides built-in analytics:

```typescript
const analytics = submitter.getAnalytics();
console.log('Submission analytics:', analytics);

// Output:
// Submission analytics: {
//   totalSubmissions: 100,
//   successfulSubmissions: 98,
//   failedSubmissions: 2,
//   averageResponseTime: 250
// }
```

<!-- TOC --><a href="#" name="use-cases"></a>
## Use Cases

1. **E-commerce Sites**: Automatically submit new product pages or updated inventory pages to ensure search engines quickly index your latest offerings.

2. **News Websites**: Instantly notify search engines about breaking news articles to improve their visibility in search results.

3. **Blogs**: Submit new blog posts as soon as they're published to accelerate their indexing and potential organic traffic.

4. **Large Websites**: Use the sitemap parsing feature to easily submit all URLs from your sitemap, ensuring your entire site stays freshly indexed.

5. **Development Workflows**: Integrate IndexNow Submitter into your CI/CD pipeline to automatically submit URLs of newly deployed or updated pages.

<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

IndexNow Submitter simplifies the process of keeping search engines up-to-date with your website's content. Whether you're managing a small blog or a large e-commerce site, this npm module provides the tools you need to ensure your content gets indexed quickly and efficiently.

By leveraging the IndexNow protocol and providing features like batch submissions, sitemap parsing, and intelligent caching, IndexNow Submitter helps you optimize your SEO efforts and potentially improve your search engine rankings.

Give IndexNow Submitter a try in your next project, and experience the benefits of faster indexing and improved search engine visibility!

---

You can find the full documentation and source code on [GitHub](https://github.com/viv1/indexnow-submitter), and the package is available on [npm](https://www.npmjs.com/package/indexnow-submitter). If you have any questions or feedback, feel free to open an issue or contribute to the project. Happy indexing!