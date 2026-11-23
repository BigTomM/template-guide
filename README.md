```
const fs = require("fs"), path = require("path");
const root = path.resolve(process.argv[2] || "");

const managedKey = "permissions.disableBypassPermissionsMode";
const managedLeaf = "disableBypassPermissionsMode";

const policyOverrides = {
};

const managedOverrides = {
};

const getter =
  /(?<![\w$])getPolicyValue\(\s*([\w$]+)\s*\)\s*\{\s*return\s+this\.policies\.get\(\s*\1\s*\)\s*;?\s*\}/g;

const patchedGetter =
  /(?<![\w$])getPolicyValue\(\s*[\w$]+\s*\)\s*\{\s*const\s+debugOverrides\s*=/g;

const send =
  /this\._sendExtensionNotification\("setClientManagedSettingsPermissions",\{permissions:([\w$]+)\},/g;

const managedMerge =
  /;\s*([\w$]+)\(this\._managedSettings,\s*([\w$]+)\.values\)\s*\|\|/g;

// 按 managedSettings.resolve 的内容匹配，不依赖混淆后的函数名。
const resolverReturn =
  /(\.resolve\(\{clientName:[\s\S]{0,900}?;if\(!([\w$]+)\)throw[\s\S]{0,300}?;)(return\s+\2(?=catch))/g;

const managedMarker = "/*__MANAGED_DEBUG_OVERRIDE__*/";
const removeMergeMarker = "/*__REMOVE_BYPASS_SETTING__*/";
const removeResolvedMarker = "/*__REMOVE_RESOLVED_BYPASS_SETTING__*/";

let changedFiles = 0;
let policySites = 0;
let sendSites = 0;
let managedSites = 0;
let resolvedSites = 0;

function walk(dir) {
  for (const f of fs.readdirSync(dir, { withFileTypes: true })) {
    const p = path.join(dir, f.name);

    if (f.isDirectory()) {
      walk(p);
      continue;
    }

    if (!f.isFile() || !f.name.toLowerCase().endsWith(".js")) continue;

    let s = fs.readFileSync(p, "utf8");
    let changed = false;

    policySites += [...s.matchAll(patchedGetter)].length;

    s = s.replace(getter, (_, name) => {
      policySites++;
      changed = true;

      return `getPolicyValue(${name}){const debugOverrides=${JSON.stringify(policyOverrides)};if(Object.hasOwn(debugOverrides,${name}))return debugOverrides[${name}];return this.policies.get(${name})}`;
    });

    s = s.replace(send, (match, perms, offset, whole) => {
      sendSites++;

      if (whole.slice(Math.max(0, offset - 120), offset)
        .includes(`delete ${perms}.disableBypassPermissionsMode;`)) {
        return match;
      }

      changed = true;
      return `delete ${perms}.disableBypassPermissionsMode;${match}`;
    });

    s = s.replace(managedMerge, (match, comparer, data, offset, whole) => {
      managedSites++;

      const before = whole.slice(Math.max(0, offset - 5000), offset);
      const addOverrides =
        !before.includes(`${managedMarker}Object.assign(${data}.values,`);
      const addDelete = !before.includes(removeMergeMarker);

      if (!addOverrides && !addDelete) return match;

      changed = true;

      const overrideCode = addOverrides
        ? `${managedMarker}Object.assign(${data}.values,${JSON.stringify(managedOverrides)});`
        : "";

      const deleteCode = addDelete
        ? `${removeMergeMarker}delete ${data}.values[${JSON.stringify(managedKey)}];`
        : "";

      return `;${overrideCode}${deleteCode}${comparer}(this._managedSettings,${data}.values)||`;
    });

    s = s.replace(resolverReturn, (match, prefix, result, ret) => {
      if (match.includes(removeResolvedMarker)) return match;

      changed = true;
      return `${prefix}${removeResolvedMarker}if(${result}.resolved?.settings?.permissions)delete ${result}.resolved.settings.permissions.${managedLeaf};${ret}`;
    });

    managedSites += s.split(removeMergeMarker).length - 1;
    resolvedSites += s.split(removeResolvedMarker).length - 1;

    if (changed) {
      fs.writeFileSync(p, s, "utf8");
      console.log("已修改：" + p);
      changedFiles++;
    }
  }
}

try {
  if (!process.argv[2] || !fs.statSync(root).isDirectory()) {
    throw Error("目录不存在");
  }

  walk(root);

  if (!policySites || !sendSites || !managedSites || !resolvedSites) {
    throw Error("未找到全部覆盖入口");
  }

  console.log(
    changedFiles
      ? `成功：修改 ${changedFiles} 个文件`
      : "成功：删除逻辑已存在，没有重复插入"
  );
} catch (e) {
  console.error("失败：" + e.message);
  process.exitCode = 1;
}
```
