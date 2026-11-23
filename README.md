# template-guide

```
const fs = require("fs"), path = require("path");
const root = path.resolve(process.argv[2] || "");
const debugOverrides = {};

const getter = /(?<![\w$])getPolicyValue\s*\(\s*([\w$]+)\s*\)\s*\{\s*return\s+this\.policies\.get\(\s*\1\s*\)\s*;?\s*\}/g;
const send = /this\._sendExtensionNotification\("setClientManagedSettingsPermissions",\{permissions:([\w$]+)\},/g;
let changed = 0;

function walk(dir) {
  for (const f of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, f.name);
    if (f.isDirectory()) walk(p);
    else if (/\.js$/i.test(p)) {
      let n = 0;
      let s = fs.readFileSync(p, "utf8");
      s = s.replace(getter, (_, name) => {
        n++;
        return `getPolicyValue(${name}){const debugOverrides=${JSON.stringify(debugOverrides)};if(Object.hasOwn(debugOverrides,${name}))return debugOverrides[${name}];return this.policies.get(${name})}`;
      });
      s = s.replace(send, (_, perms) => {
        n++;
        return `delete ${perms}.disableBypassPermissionsMode;this._sendExtensionNotification("setClientManagedSettingsPermissions",{permissions:${perms}},`;
      });
      if (n) {
        fs.writeFileSync(p, s, "utf8");
        console.log("已修改：" + p);
        changed++;
      }
    }
  }
}

try {
  if (!process.argv[2] || !fs.statSync(root).isDirectory()) throw Error("目录不存在");
  walk(root);
  if (!changed) throw Error("没有找到匹配代码");
  console.log(`成功：修改 ${changed} 个文件`);
} catch (e) {
  console.error("失败：" + e.message);
  process.exitCode = 1;
}
```

```
node .\patch.js 'D:\xxx\vsc'
```
