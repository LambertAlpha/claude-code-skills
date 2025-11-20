---
name: dynamic-web-extractor
description: Universal web content extractor for JavaScript-heavy websites (SPAs). ALWAYS use when: (1) WebFetch returns "JavaScript is not available", (2) User provides URLs from Twitter/X, Reddit, Medium, LinkedIn, Instagram, cryptocurrency exchanges (Binance, Hyperliquid, Uniswap, etc.), or ANY Web3/DeFi platforms, (3) User asks to "extract", "scrape", or "analyze" a website, (4) You see a modern web app URL that likely needs JS rendering. NEVER use puppeteer_screenshot - ALWAYS use puppeteer_evaluate for text extraction.
---

# Dynamic Web Content Extractor

## ⚠️ 强制决策树（必读！）

**在处理任何 URL 之前，严格遵循以下决策流程**：

```
收到 URL 请求
  │
  ├─ Step 1: 域名识别
  │   ├─ 是 Twitter/X, Reddit, Medium, LinkedIn? → 跳到 Step 3
  │   ├─ 是加密货币/DeFi 网站? (包含: trade, swap, dex, exchange, crypto, defi) → 跳到 Step 3
  │   └─ 其他 → 继续 Step 2
  │
  ├─ Step 2: 快速测试 WebFetch
  │   ├─ WebFetch 成功且返回完整内容? → ✅ 使用 WebFetch，任务完成
  │   ├─ 返回 "JavaScript is not available"? → 继续 Step 3
  │   └─ 返回内容很少（<500字符）? → 继续 Step 3
  │
  └─ Step 3: 使用 Puppeteer（本 Skill）
      ├─ ❌ 绝对禁止：puppeteer_screenshot（有 bug，总是失败）
      ├─ ✅ 必须使用：puppeteer_navigate + puppeteer_evaluate
      │
      ├─ 3.1: 导航到页面
      │   mcp__puppeteer__puppeteer_navigate(url)
      │
      ├─ 3.2: 执行智能提取脚本
      │   ├─ 检测 Cloudflare 验证? → 见"反爬虫对策"章节
      │   ├─ 检测登录要求? → 见"需要认证"章节
      │   ├─ 使用域名识别 → 应用对应模板
      │   └─ 否则使用通用模板
      │
      └─ 3.3: 返回结构化数据
          └─ 如果所有选择器失败 → 返回 document.body.innerText 兜底
```

### 🚨 常见错误（不要犯！）

| 错误行为 | 为什么错 | 正确做法 |
|---------|---------|---------|
| 直接用 `puppeteer_screenshot` | 工具有 bug，图片格式声明错误 | 用 `puppeteer_evaluate` 提取文本 |
| 不等待页面加载就提取 | 内容还没渲染完 | `await new Promise(r => setTimeout(r, 5000))` |
| 只用一个选择器 | 网站改版就失效 | 用 3-5 个备选选择器 |
| 遇到 Cloudflare 就放弃 | 可以等待或重试 | 见下方"反爬虫对策" |
| 看到加密货币网站用 WebFetch | 100% 是 SPA | 直接用 Puppeteer |

## 概述

这是一个**通用的动态网页内容提取工具**，专门用于处理需要 JavaScript 渲染的现代网站（SPA - Single Page Applications）。

### 适用场景

- ✅ Twitter/X 推文
- ✅ Reddit 帖子
- ✅ Medium 文章
- ✅ LinkedIn 动态
- ✅ Instagram 帖子（需要登录）
- ✅ **加密货币交易平台**（Binance, Hyperliquid, Uniswap, Coinbase, etc.）
- ✅ **DeFi/Web3 应用**（几乎 100% 是 SPA）
- ✅ 任何用 React/Vue/Angular 构建的网站
- ✅ 任何 WebFetch 抓取失败的网站

### 不适用场景

- ❌ 静态网站（如维基百科）→ 直接用 WebFetch 更快
- ❌ 需要复杂交互的网站（如滚动加载）→ 需要自定义脚本
- ❌ 需要登录验证的内容 → 需要 cookies/session

## 🛡️ 反爬虫对策（Cloudflare/验证码处理）

### 检测反爬虫机制

在提取脚本中，首先检测是否遇到反爬虫：

```javascript
(async () => {
  // 等待页面加载
  await new Promise(resolve => setTimeout(resolve, 5000));

  // 检测 Cloudflare 验证
  const bodyText = document.body.innerText;
  const isCloudflare =
    bodyText.includes('Cloudflare') ||
    bodyText.includes('驗證您是人類') ||
    bodyText.includes('Verify you are human') ||
    bodyText.includes('Just a moment') ||
    bodyText.includes('Checking your browser');

  // 检测登录墙
  const isLoginRequired =
    bodyText.includes('Sign in') ||
    bodyText.includes('Log in') ||
    bodyText.includes('登入') ||
    bodyText.includes('登录') ||
    document.querySelector('[type="password"]') !== null;

  if (isCloudflare) {
    return {
      error: 'CLOUDFLARE_CHALLENGE',
      message: '检测到 Cloudflare 人机验证',
      suggestion: '建议: (1) 等待更长时间自动通过, (2) 使用官方 API 代替, (3) 考虑使用代理服务',
      rawText: bodyText.slice(0, 1000)
    };
  }

  if (isLoginRequired) {
    return {
      error: 'LOGIN_REQUIRED',
      message: '该内容需要登录才能访问',
      suggestion: '建议: (1) 提供登录凭证, (2) 使用官方 API, (3) 寻找公开的数据源',
      rawText: bodyText.slice(0, 1000)
    };
  }

  // 继续正常提取...
})();
```

### Cloudflare 绕过策略

#### 策略 1：等待自动通过（最常用）

```javascript
// 增加等待时间，让 Cloudflare 自动验证通过
await new Promise(resolve => setTimeout(resolve, 10000)); // 等待 10 秒

// 再次检查
if (document.body.innerText.includes('Cloudflare')) {
  // 仍然被拦截，返回错误
  return { error: 'CLOUDFLARE_TIMEOUT' };
}
```

#### 策略 2：重试机制

```javascript
// 在主脚本中实现重试
async function extractWithRetry(url, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    await mcp__puppeteer__puppeteer_navigate(url);
    await new Promise(r => setTimeout(r, 5000 * (i + 1))); // 逐渐增加等待时间

    const result = await mcp__puppeteer__puppeteer_evaluate(extractScript);

    if (!result.error) {
      return result; // 成功
    }

    console.log(`尝试 ${i + 1}/${maxRetries} 失败，重试中...`);
  }

  return { error: 'MAX_RETRIES_EXCEEDED' };
}
```

#### 策略 3：建议用户使用官方 API（最佳方案）

```javascript
if (hostname.includes('hyperliquid') || hostname.includes('binance') || hostname.includes('coinbase')) {
  return {
    error: 'USE_API_INSTEAD',
    message: '检测到加密货币交易平台',
    suggestion: `建议使用官方 API 代替网页抓取：

    Hyperliquid API: https://api.hyperliquid.xyz/info
    Binance API: https://api.binance.com/api/v3/
    Coinbase API: https://api.coinbase.com/v2/

    原因:
    1. API 更稳定、更快
    2. 没有反爬虫限制
    3. 数据格式标准化
    4. 避免法律风险`,
    rawText: bodyText.slice(0, 500)
  };
}
```

### 何时放弃并建议替代方案

**立即放弃的情况**：
1. ✅ 检测到官方 API 存在 → 建议用户改用 API
2. ✅ 多次重试仍失败 → 返回错误并说明原因
3. ✅ 检测到需要复杂交互（如拖动验证码）→ 超出本 Skill 能力

**继续尝试的情况**：
1. ✅ Cloudflare 自动验证（等待即可）
2. ✅ 简单的延迟加载（增加等待时间）
3. ✅ 临时性网络问题（重试）

## 核心原理

### 判断网站类型

| 网站类型 | 特征 | 推荐工具 |
|---------|------|---------|
| **静态网站** | HTML 直接包含内容 | WebFetch |
| **动态网站** | 需要 JS 渲染 | **本 Skill (Puppeteer)** |
| **需要登录** | 返回登录页 | Puppeteer + Cookies |

**快速判断方法**：
```
打开浏览器 DevTools → 禁用 JavaScript → 刷新
- 内容还在 → 静态网站
- 内容消失 → 动态网站（需要本 Skill）
```

### 为什么需要 Puppeteer？

| 方法 | 支持 JS | 速度 | 准确度 | 适用场景 |
|------|---------|------|--------|----------|
| WebFetch | ❌ | 快 | 低（动态站） | 静态网站 |
| puppeteer_screenshot | ✅ | 慢 | 低（OCR）| 需要布局信息 |
| **puppeteer_evaluate** | ✅ | 中 | **高** | **动态网站（推荐）** |

## 使用方法

### 1. 基础工作流

```javascript
// Step 1: 导航到目标页面
mcp__puppeteer__puppeteer_navigate(url: "https://example.com/post/123")

// Step 2: 执行提取脚本
mcp__puppeteer__puppeteer_evaluate(script: extractionScript)
```

### 2. 通用提取模板

```javascript
(async () => {
  // ==================== 配置区 ====================
  const config = {
    // 等待时间（毫秒）- 根据网站加载速度调整
    waitTime: 5000,

    // 主内容选择器（按优先级排序）
    contentSelectors: [
      'article',                    // 通用文章容器
      'main',                       // 主内容区域
      '[role="main"]',              // ARIA 主内容
      '.post-content',              // 常见类名
      '#content'                    // 常见 ID
    ],

    // 标题选择器
    titleSelectors: [
      'h1',
      'article h1',
      '[role="heading"]',
      '.post-title'
    ],

    // 作者选择器
    authorSelectors: [
      '[rel="author"]',
      '.author-name',
      'a[href*="/user/"]',
      '[data-testid="author"]'
    ],

    // 时间戳选择器
    timestampSelectors: [
      'time',
      '[datetime]',
      '.publish-date',
      '[data-testid="timestamp"]'
    ]
  };

  // ==================== 等待加载 ====================
  await new Promise(resolve => setTimeout(resolve, config.waitTime));

  // ==================== 提取函数 ====================
  const extractBySelectors = (selectors) => {
    for (const selector of selectors) {
      const element = document.querySelector(selector);
      if (element && element.innerText.trim()) {
        return element.innerText.trim();
      }
    }
    return '';
  };

  const extractAttribute = (selectors, attribute) => {
    for (const selector of selectors) {
      const element = document.querySelector(selector);
      if (element) {
        return element.getAttribute(attribute) || element.innerText.trim();
      }
    }
    return '';
  };

  // ==================== 执行提取 ====================
  const data = {
    title: extractBySelectors(config.titleSelectors),
    content: extractBySelectors(config.contentSelectors),
    author: extractBySelectors(config.authorSelectors),
    timestamp: extractAttribute(config.timestampSelectors, 'datetime'),
    url: window.location.href,

    // 元数据
    meta: {
      description: document.querySelector('meta[name="description"]')?.content || '',
      keywords: document.querySelector('meta[name="keywords"]')?.content || '',
      ogTitle: document.querySelector('meta[property="og:title"]')?.content || '',
      ogImage: document.querySelector('meta[property="og:image"]')?.content || ''
    },

    // 兜底：如果所有选择器失败，返回页面主要文本
    fallbackText: ''
  };

  // 如果主内容为空，使用兜底策略
  if (!data.content) {
    data.fallbackText = document.body.innerText.slice(0, 5000);
  }

  return data;
})();
```

## 预配置网站模板

### Twitter/X 专用配置

```javascript
// 检测到 x.com 或 twitter.com 时使用
const twitterExtract = (async () => {
  await new Promise(resolve => setTimeout(resolve, 5000));

  const data = {
    text: '',
    author: '',
    username: '',
    timestamp: '',
    metrics: { likes: 0, retweets: 0, replies: 0 }
  };

  // 提取推文文本
  const textSelectors = [
    '[data-testid="tweetText"]',
    'article div[lang]'
  ];
  for (const sel of textSelectors) {
    const el = document.querySelector(sel);
    if (el?.innerText) {
      data.text = el.innerText;
      break;
    }
  }

  // 提取作者
  const authorEl = document.querySelector('[data-testid="User-Name"] span');
  if (authorEl) data.author = authorEl.innerText;

  // 提取时间
  const timeEl = document.querySelector('time');
  if (timeEl) data.timestamp = timeEl.getAttribute('datetime');

  return data;
})();
```

### Reddit 专用配置

```javascript
// 检测到 reddit.com 时使用
const redditExtract = (async () => {
  await new Promise(resolve => setTimeout(resolve, 3000));

  return {
    title: document.querySelector('shreddit-post h1')?.innerText || '',
    content: document.querySelector('[slot="text-body"]')?.innerText || '',
    author: document.querySelector('shreddit-post')?.getAttribute('author') || '',
    subreddit: document.querySelector('shreddit-post')?.getAttribute('subreddit-prefixed-name') || '',
    score: document.querySelector('[slot="up-vote"] + faceplate-number')?.innerText || '0',
    comments: document.querySelector('[slot="comment-count"]')?.innerText || '0'
  };
})();
```

### Medium 专用配置

```javascript
// 检测到 medium.com 时使用
const mediumExtract = (async () => {
  await new Promise(resolve => setTimeout(resolve, 4000));

  return {
    title: document.querySelector('h1')?.innerText || '',
    content: document.querySelector('article section')?.innerText || '',
    author: document.querySelector('a[rel="author"]')?.innerText || '',
    readTime: document.querySelector('[data-testid="storyReadTime"]')?.innerText || '',
    tags: Array.from(document.querySelectorAll('a[href*="/tag/"]')).map(a => a.innerText)
  };
})();
```

### LinkedIn 专用配置

```javascript
// 检测到 linkedin.com 时使用
const linkedinExtract = (async () => {
  await new Promise(resolve => setTimeout(resolve, 6000));

  return {
    content: document.querySelector('.feed-shared-update-v2__description')?.innerText || '',
    author: document.querySelector('.update-components-actor__name')?.innerText || '',
    timestamp: document.querySelector('time')?.getAttribute('datetime') || '',
    reactions: document.querySelector('.social-details-social-counts__reactions-count')?.innerText || '0'
  };
})();
```

### 智能网站识别脚本

```javascript
(async () => {
  const hostname = window.location.hostname;

  // 根据域名选择提取策略
  if (hostname.includes('twitter.com') || hostname.includes('x.com')) {
    // Twitter 提取逻辑
    await new Promise(r => setTimeout(r, 5000));
    return {
      site: 'twitter',
      text: document.querySelector('[data-testid="tweetText"]')?.innerText || '',
      author: document.querySelector('[data-testid="User-Name"] span')?.innerText || '',
      timestamp: document.querySelector('time')?.getAttribute('datetime') || ''
    };
  } else if (hostname.includes('reddit.com')) {
    // Reddit 提取逻辑
    await new Promise(r => setTimeout(r, 3000));
    return {
      site: 'reddit',
      title: document.querySelector('shreddit-post h1')?.innerText || '',
      content: document.querySelector('[slot="text-body"]')?.innerText || '',
      author: document.querySelector('shreddit-post')?.getAttribute('author') || ''
    };
  } else if (hostname.includes('medium.com')) {
    // Medium 提取逻辑
    await new Promise(r => setTimeout(r, 4000));
    return {
      site: 'medium',
      title: document.querySelector('h1')?.innerText || '',
      content: document.querySelector('article section')?.innerText || '',
      author: document.querySelector('a[rel="author"]')?.innerText || ''
    };
  } else {
    // 通用提取逻辑
    await new Promise(r => setTimeout(r, 5000));
    return {
      site: 'generic',
      title: document.querySelector('h1')?.innerText || '',
      content: document.querySelector('article, main, [role="main"]')?.innerText || '',
      fallbackText: document.body.innerText.slice(0, 5000)
    };
  }
})();
```

## 高级技巧

### 处理无限滚动

```javascript
// 滚动加载更多内容
let lastHeight = document.body.scrollHeight;
for (let i = 0; i < 5; i++) {
  window.scrollTo(0, document.body.scrollHeight);
  await new Promise(r => setTimeout(r, 2000));
  if (document.body.scrollHeight === lastHeight) break;
  lastHeight = document.body.scrollHeight;
}
```

### 等待特定元素加载

```javascript
// 等待关键元素出现
await new Promise((resolve) => {
  const interval = setInterval(() => {
    if (document.querySelector('[data-testid="content"]')) {
      clearInterval(interval);
      resolve();
    }
  }, 100);
  setTimeout(() => { clearInterval(interval); resolve(); }, 10000);
});
```

### 提取所有链接和图片

```javascript
// 提取链接
const links = Array.from(document.querySelectorAll('a'))
  .map(a => ({ text: a.innerText, href: a.href }))
  .filter(l => l.href && !l.href.startsWith('javascript:'));

// 提取图片
const images = Array.from(document.querySelectorAll('img'))
  .map(img => ({ src: img.src, alt: img.alt }))
  .filter(i => i.src && !i.src.startsWith('data:'));
```

## 最佳实践

### ✅ 应该这样做

1. **先判断网站类型再选择工具**
   ```javascript
   // 先尝试 WebFetch (快)
   // 如果看到 "JavaScript is not available" → 改用 Puppeteer
   ```

2. **使用渐进式等待策略**
   ```javascript
   // 根据网站速度调整
   // 快速网站 (Reddit): 3s
   // 中速网站 (Twitter): 5s
   // 慢速网站 (LinkedIn): 6-7s
   ```

3. **多重选择器 + 兜底机制**
   ```javascript
   const selectors = [specific, lessSpecific, generic, fallback];
   // 如果全部失败，返回 document.body.innerText
   ```

4. **提取元数据作为补充**
   ```javascript
   // Open Graph, Twitter Card, JSON-LD 都有价值
   meta: {
     ogTitle: document.querySelector('meta[property="og:title"]')?.content,
     ogImage: document.querySelector('meta[property="og:image"]')?.content
   }
   ```

5. **使用网站检测自动选择策略**
   ```javascript
   if (hostname.includes('twitter.com')) {
     // 使用 Twitter 配置
   } else {
     // 使用通用配置
   }
   ```

### ❌ 避免的错误

1. **不要对所有网站用同一个等待时间** - 根据网站速度调整
2. **不要只提取可见文本** - 也要提取 data-* 属性、aria-label 等
3. **不要忽略错误处理** - 总是提供 fallbackText
4. **不要在循环中重复查询 DOM** - 使用 map + filter 优化

## 故障排查

### 问题矩阵

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| 返回空内容 | 页面未加载完 | 增加 waitTime 到 10 秒 |
| 返回 "JavaScript not available" | 用了 WebFetch | 改用 Puppeteer |
| 选择器失效 | 网站改版 | 用 DevTools 找新选择器 |
| 提取内容不完整 | 需要滚动加载 | 添加滚动逻辑 |
| 提取到登录页 | 需要认证 | 添加 cookies（超出本 Skill 范围）|
| 截图格式错误 | Puppeteer bug | 改用 evaluate 提取文本 |

### 调试技巧

```javascript
// 开启调试模式 - 返回更多信息
const debug = {
  url: window.location.href,
  hostname: window.location.hostname,
  htmlPreview: document.documentElement.outerHTML.slice(0, 500),
  foundSelectors: config.contentSelectors.map(s => ({
    selector: s,
    found: !!document.querySelector(s),
    preview: document.querySelector(s)?.innerText.slice(0, 50)
  })),
  pageHeight: document.body.scrollHeight,
  loadTime: performance.now()
};

return { ...data, debug };
```

## 快速参考

### 网站类型速查表

| 网站 | 等待时间 | 主要选择器 | 特殊处理 |
|------|---------|-----------|---------|
| Twitter/X | 5s | `[data-testid="tweetText"]` | 多重选择器 |
| Reddit | 3s | `shreddit-post, [slot="text-body"]` | 新旧版本兼容 |
| Medium | 4s | `article section` | 无 |
| LinkedIn | 6s | `.feed-shared-update-v2__description` | 需要长时间等待 |
| Instagram | 5s | `article` | 可能需要登录 |
| **通用** | 5s | `article, main, [role="main"]` | **推荐起点** |

### 常见选择器模式

```javascript
// 语义化 HTML5
['article', 'main', 'section', '[role="main"]']

// 通用类名
['.post-content', '.article-body', '.entry-content']

// 通用 ID
['#content', '#main-content', '#post-content']

// 数据属性
['[data-testid="content"]', '[data-content]']

// ARIA 属性
['[role="article"]', '[aria-label*="content"]']
```

### 决策树：选择合适的方法

```
用户提供 URL
  ├─ 是否是已知网站 (Twitter/Reddit/Medium)?
  │   ├─ 是 → 使用预配置脚本
  │   └─ 否 → 继续
  │
  ├─ 先尝试 WebFetch
  │   ├─ 成功获取内容 → 完成
  │   └─ 返回"JavaScript not available" → 继续
  │
  └─ 使用 Puppeteer
      ├─ 使用智能识别脚本（检测域名）
      └─ 或使用通用提取模板
```

## 技术原理总结

1. **现代网站多为 SPA** → 需要 JS 支持
2. **内容异步加载** → 必须等待渲染完成
3. **网站结构多变** → 多重选择器 + 兜底策略
4. **文本提取优于 OCR** → 直接读 DOM 更准确
5. **元数据很有价值** → Open Graph / JSON-LD 补充信息

## 相关工具和资源

- **Puppeteer 文档**: https://pptr.dev/
- **选择器测试**: 浏览器 DevTools Console
- **网站结构分析**: Chrome DevTools → Elements 面板
- **性能分析**: Chrome DevTools → Performance 面板
- **CSS 选择器参考**: https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors

## 扩展方向

想要支持新网站？只需：
1. 用 DevTools 分析网站结构
2. 找出内容、标题、作者的选择器
3. 在"预配置网站模板"添加新配置
4. 在"智能网站识别脚本"添加域名检测

---

## 📋 更新日志

### v1.1.0 (2025-11-19)
**重大更新**：根据 Hyperliquid 提取失败的实战经验改进

**新增功能**：
- ✅ 强制决策树：明确什么时候用什么工具
- ✅ Cloudflare/反爬虫检测和对策
- ✅ 加密货币/DeFi 网站自动识别
- ✅ 错误类型分类（CLOUDFLARE_CHALLENGE, LOGIN_REQUIRED, USE_API_INSTEAD）
- ✅ 建议用户使用官方 API 而非爬虫（适用于加密货币平台）

**改进**：
- ✅ description 更明确：列出所有应该触发的场景
- ✅ 强调"绝对禁止 puppeteer_screenshot"
- ✅ 添加常见错误对照表

**修复**：
- ✅ 之前没有自动识别加密货币网站
- ✅ 之前遇到 Cloudflare 就失败，现在提供多种对策
- ✅ 之前没有建议用户改用 API

### v1.0.0 (2025-11-19)
- ✅ 初始版本
- ✅ 支持 Twitter/X, Reddit, Medium, LinkedIn
- ✅ 通用提取模板
- ✅ 智能网站识别

---

**当前版本**: v1.1.0
**最后更新**: 2025-11-19
**测试环境**: macOS, Puppeteer MCP Server
**支持网站**: Twitter/X, Reddit, Medium, LinkedIn, 加密货币交易所, DeFi/Web3 应用, 通用 SPA
**已知限制**:
- 需要登录的内容无法提取（需要 cookies）
- `puppeteer_screenshot` 有格式 bug，**禁止使用**
- Cloudflare 拖动验证码无法绕过（建议改用官方 API）
- 某些网站的 API 限流可能导致失败（需要添加 rate limiting）
