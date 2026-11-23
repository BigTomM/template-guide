# template-guide

```
const fs = require("fs"), path = require("path");
const root = path.resolve(process.argv[2] || "");
const debugOverrides = {
  // ChatMCP: "registry",
  // ChatToolsAutoApprove: true,
};
const re = /(?<![\w$])getPolicyValue\s*\(\s*([\w$]+)\s*\)\s*\{\s*return\s+this\.policies\.get\(\s*\1\s*\)\s*;?\s*\}/g;
let changed = 0;

function walk(dir) {
  for (const f of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, f.name);
    if (f.isDirectory()) walk(p);
    else if (/\.js$/i.test(p)) {
      const s = fs.readFileSync(p, "utf8");
      let n = 0;
      const t = s.replace(re, (_, name) => {
        n++;
        return `getPolicyValue(${name}){const debugOverrides=${JSON.stringify(debugOverrides)};if(Object.hasOwn(debugOverrides,${name}))return debugOverrides[${name}];return this.policies.get(${name})}`;
      });
      if (n) {
        const bak = p + ".policy-debug.bak";
        if (!fs.existsSync(bak)) fs.copyFileSync(p, bak);
        fs.writeFileSync(p, t, "utf8");
        changed++;
      }
    }
  }
}

try {
  if (!process.argv[2] || !fs.statSync(root).isDirectory()) throw Error("目录不存在");
  walk(root);
  if (!changed) throw Error("没有找到符合条件的 getter");
  console.log(`成功：修改了 ${changed} 个 JS 文件`);
} catch (e) {
  console.error("失败：" + e.message);
  process.exitCode = 1;
}
```

```
node .\patch.js 'D:\xxx\vsc'
```
