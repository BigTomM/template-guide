# template-guide

```
const fs = require("fs"), path = require("path");
const root = path.resolve(process.argv[2] || "");

// 如果文件已经有前一版覆盖，会保留原值；在干净文件上运行时，把已有 policy 覆盖值填到这里。
const policyOverrides = {

};

const managedOverrides = {

};

const getter = /(?<![\w$])getPolicyValue\s*\(\s*([\w$]+)\s*\)\s*\{\s*return\s+this\.policies\.get\(\s*\1\s*\)\s*;?\s*\}/g;
const patchedGetter = /(?<![\w$])getPolicyValue\s*\(\s*[\w$]+\s*\)\s*\{\s*const\s+debugOverrides\s*=/g;
const send = /this\._sendExtensionNotification\("setClientManagedSettingsPermissions",\{permissions:([\w$]+)\},/g;
const managedMerge = /;\s*([\w$]+)\s*\(\s*this\._managedSettings\s*,\s*([\w$]+)\.values\s*\)\s*\|\|/g;
const managedMarker = "/*__MANAGED_DEBUG_OVERRIDE__*/";

let changedFiles = 0, policySites = 0, sendSites = 0, managedSites = 0;

function walk(dir) {
  for (const f of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, f.name);
    if (f.isDirectory()) {
      walk(p);
      continue;
    }
    if (!f.isFile() || !f.name.toLowerCase().endsWith(".js")) continue;

    let s = fs.readFileSync(p, "utf8"), changed = false;
    policySites += [...s.matchAll(patchedGetter)].length;
    s = s.replace(getter, (_, name) => {
      policySites++;
      changed = true;
      return `getPolicyValue(${name}){const debugOverrides=${JSON.stringify(policyOverrides)};if(Object.hasOwn(debugOverrides,${name}))return debugOverrides[${name}];return this.policies.get(${name})}`;
    });

    s = s.replace(send, (match, perms, offset, whole) => {
      sendSites++;
      if (whole.slice(Math.max(0, offset - 100), offset)
        .includes(`delete ${perms}.disableBypassPermissionsMode;`)) return match;
      changed = true;
      return `delete ${perms}.disableBypassPermissionsMode;${match}`;
    });

    managedSites += s.split(managedMarker).length - 1;
    s = s.replace(managedMerge, (_, comparer, data) => {
      managedSites++;
      changed = true;
      return `;${managedMarker}Object.assign(${data}.values,${JSON.stringify(managedOverrides)});${comparer}(this._managedSettings,${data}.values)||`;
    });

    if (changed) {
      fs.writeFileSync(p, s, "utf8");
      console.log("已修改：" + p);
      changedFiles++;
    }
  }
}

try {
  if (!process.argv[2] || !fs.statSync(root).isDirectory()) throw Error("目录不存在");
  walk(root);
  if (!policySites || !sendSites || !managedSites) throw Error("未找到全部覆盖入口");
  console.log(changedFiles
    ? `成功：修改 ${changedFiles} 个文件`
    : "成功：覆盖代码已存在，没有重复插入");
} catch (e) {
  console.error("失败：" + e.message);
  process.exitCode = 1;
}
```

```
node .\patch.js 'D:\xxx\vsc'
```
