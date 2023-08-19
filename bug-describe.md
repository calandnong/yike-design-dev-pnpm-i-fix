### 版本
* 操作系统版本：macos
* 项目版本：yike-design-dev/monorepo-dev

### 重现步骤
1. 首次进行git clone https://github.com/ecaps1038/yike-design-dev -b monorepo-dev
2. cd到项目文件根路径，执行pnpm i
3. 显示以下报错，请注意【告警 - WARN】部分和【错误】部分
```bash
Scope: all 5 workspace projects
Lockfile is up-to-date, resolution step is skipped
Packages: +796
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Packages are hard linked from the content-addressable store to the virtual store.
  Content-addressable store is at: /.pnpm-store/v3
  Virtual store is at:             node_modules/.pnpm
node_modules/.pnpm/vue-demi@0.14.5_vue@3.3.4/node_modules/vue-demi: Running postinstall script, done in 74ms
Progress: resolved 796, reused 794, downloaded 0, added 796, done
node_modules/.pnpm/esbuild@0.18.20/node_modules/esbuild: Running postinstall script, done in 156ms
 WARN  Failed to create bin at /Projects/ui-components/yike-design-dev/packages/yike-design-ui/node_modules/.bin/yike-build. The source file at /Projects/ui-components/yike-design-dev/build/dist/index.js does not exist.

dependencies:
+ vue 3.3.4

devDependencies:
+ @testing-library/vue 7.0.0
+ @types/node 18.17.6
+ @typescript-eslint/eslint-plugin 6.4.0
+ @typescript-eslint/parser 6.4.0
+ @vitejs/plugin-vue 4.3.1
+ @vitejs/plugin-vue-jsx 3.0.2
+ eslint 8.47.0
+ eslint-config-prettier 8.10.0
+ eslint-import-resolver-typescript 3.6.0
+ eslint-plugin-import 2.28.1
+ eslint-plugin-unused-imports 2.0.0
+ eslint-plugin-vue 9.17.0
+ husky 8.0.3
+ less 4.2.0
+ lint-staged 13.3.0
+ postcss 8.4.28
+ postcss-html 1.5.0
+ postcss-less 6.0.0
+ prettier 2.8.8
+ standard-version 9.5.0
+ stylelint 15.10.3
+ stylelint-config-recommended-vue 1.5.0
+ stylelint-config-standard-vue 1.0.0
+ stylelint-less 1.0.8
+ stylelint-order 6.0.3
+ typescript 5.1.6
+ vite 4.4.9
+ vue-tsc 1.8.8

. postinstall$ npx husky install
│ husky - Git hooks installed
└─ Done in 1s
. prepare$ pnpm --filter @yike/build build && pnpm gen:icon
│ > @yike/build@0.0.1 build /Projects/ui-components/yike-design-dev/build
│ > tsc
│ > yike-design-dev@0.0.1 gen:icon /Projects/ui-components/yike-design-dev
│ > pnpm --filter yike-design-ui gen:icon
│ > yike-design-ui@ gen:icon /Projects/ui-components/yike-design-dev/packages/yike-design-ui
│ > yike-build icongen
│ sh: yike-build: command not found
│ undefined
│ /Projects/ui-components/yike-design-dev/packages/yike-design-ui:
│  ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL  yike-design-ui@ gen:icon: `yike-build icongen`
│ spawn ENOENT
│  ELIFECYCLE  Command failed with exit code 1.
└─ Failed in 2.2s
 ELIFECYCLE  Command failed with exit code 1.
```
4. 错误后重新执行pnpm i执行成功

### 期望的行为
执行pnpm i即可安装依赖成功

### 实际的行为
执行pnpm i会出现报错，第二次截图才可以成功

### 附加信息（截图/代码片段等）
最小可重现demo：
仓库地址：https://github.com/calandnong/yike-design-dev-pnpm-i-fix
重新demo分支：master
修复方案分支：bug-fix/fix-build-commond-fail
原因分析：
**告警（Failed to create bin）的原因：**
依赖安装时不同配置的加载顺序导致，会先去处理bin部分的内容解析引用保存到内存中，处理完成后才会进入pnpm的钩子事件的执行。
具体原因如下：
在build/package.json中，提前设置了以下内容：
```json
  "bin": {
    "yike-build": "dist/index.js"
  },
```
pnpm会在安装依赖时会先去扫描所有包的package.json中的bin进行注册，然后，将这个命令注册到node_modules下的.bin目录下，然后会此时进行判断此文件是否存在，此处因为时机比根路径的下的prepare钩子执行时机要早，dist文件尚未生成，因此会告警

**报错（Command failed with exit code 1）的原因：**
由于前面的bin注册前提，此次pnpm i的进程运行时，已经在内存中扫描注册结束了bin命令，所以此处只会读取本次运行内存中的bin指向，因为"yike-build": "dist/index.js"中的dist/index.js不存在，所以这里这个命令会是 undefined，因此执行出错，而第二次执行为什么成功？原因是因为此次再次执行了pnpm i，且因第一次已经生成了dist文件，因此本次运行内存中的bin指向是ok的，所以可以被安装成功。

命令位置：
根路径/package.json：
```json
  "prepare": "pnpm --filter @yike/build build && pnpm gen:icon",
```

**修复方案：**
1.新增文件build/bin/index.js，内容如下：
```javascript
function run() {
  return import('../dist/index.js');
}

run();
```
2.将build/package.json中，修改为以下内容：
```diff
  "bin": {
-    "yike-build": "dist/index.js"
+    "yike-build": "bin/index.js"
  },
```
3.删除所有的node_modules，运行测试即可