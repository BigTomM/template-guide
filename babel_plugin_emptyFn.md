const { declare } = require('@babel/helper-plugin-utils');

module.exports = declare((api, options) => {
  api.assertVersion(7);

  const { 
    marker = '@need-emty',
    preserveComments = true,
    logTransformed = false 
  } = options;

  return {
    name: 'empty-functions-advanced',
    visitor: {
      // 处理函数声明
      FunctionDeclaration(path) {
        this.handleFunction(path, api, marker, preserveComments, logTransformed);
      },
      
      // 处理函数表达式
      FunctionExpression(path) {
        this.handleFunction(path, api, marker, preserveComments, logTransformed);
      },
      
      // 处理箭头函数
      ArrowFunctionExpression(path) {
        this.handleFunction(path, api, marker, preserveComments, logTransformed);
      },
      
      // 处理对象方法
      ObjectMethod(path) {
        this.handleFunction(path, api, marker, preserveComments, logTransformed);
      },
      
      // 处理类方法
      ClassMethod(path) {
        this.handleFunction(path, api, marker, preserveComments, logTransformed);
      }
    },
    
    handleFunction(path, api, marker, preserveComments, logTransformed) {
      const { node } = path;
      
      // 获取函数前的注释
      let leadingComments = node.leadingComments || [];
      
      // 如果函数本身没有注释，检查父节点的注释
      if (leadingComments.length === 0) {
        const parent = path.parent;
        if (parent && parent.leadingComments) {
          leadingComments = parent.leadingComments;
        }
      }
      
      // 检查是否有指定的标记
      const hasMarker = leadingComments.some(comment => 
        comment.value.includes(marker)
      );
      
      if (hasMarker) {
        const functionName = node.id ? node.id.name : '<anonymous>';
        
        if (logTransformed) {
          console.log(`Transforming function: ${functionName}`);
        }
        
        // 保存原始位置信息
        const originalLoc = node.loc;
        const originalStart = node.start;
        const originalEnd = node.end;
        
        // 创建空函数体
        let emptyBody;
        
        if (path.isArrowFunctionExpression()) {
          // 箭头函数特殊处理
          if (node.body.type === 'BlockStatement') {
            emptyBody = api.types.blockStatement([]);
          } else {
            // 如果原来是表达式，转换为块语句
            emptyBody = api.types.blockStatement([]);
          }
        } else {
          emptyBody = api.types.blockStatement([]);
        }
        
        // 保持原始位置信息以维护sourcemap准确性
        if (originalLoc) {
          emptyBody.loc = {
            start: originalLoc.start,
            end: originalLoc.end
          };
        }
        
        if (originalStart !== undefined) {
          emptyBody.start = originalStart;
        }
        
        if (originalEnd !== undefined) {
          emptyBody.end = originalEnd;
        }
        
        // 替换函数体
        node.body = emptyBody;
        
        // 保留注释
        if (preserveComments) {
          node.leadingComments = leadingComments;
        }
        
        // 标记节点已被转换，避免重复处理
        node._transformed = true;
      }
    }
  };
});

module.exports = {
  plugins: [
    ['./babel-plugin-empty-functions-advanced.js', {
      marker: '@need-emty',           // 自定义标记字符串
      preserveComments: true,         // 保留注释
      logTransformed: true            // 记录转换的函数
    }]
  ],
  // 启用sourcemap支持
  sourceMaps: true,
  // 保留注释以维护sourcemap准确性
  comments: true
};

// 示例输入文件

// @need-emty
function processData(data) {
  console.log('Processing data:', data);
  const result = data.map(item => item * 2);
  return result;
}

// @need-emty
const calculateSum = (numbers) => {
  return numbers.reduce((sum, num) => sum + num, 0);
};

// 这个函数不会被转换
function keepOriginal() {
  console.log('This function will remain unchanged');
}

class MyClass {
  // @need-emty
  processMethod(data) {
    console.log('Processing in method:', data);
    return data.filter(item => item > 0);
  }
  
  // 这个方法不会被转换
  normalMethod() {
    return 'normal';
  }
}

const obj = {
  // @need-emty
  transform(input) {
    console.log('Transforming:', input);
    return input.toUpperCase();
  }
};

ouput.js
// 转换后的输出

// @need-emty
function processData(data) {}

// @need-emty
const calculateSum = (numbers) => {};

// 这个函数不会被转换
function keepOriginal() {
  console.log('This function will remain unchanged');
}

class MyClass {
  // @need-emty
  processMethod(data) {}
  
  // 这个方法不会被转换
  normalMethod() {
    return 'normal';
  }
}

const obj = {
  // @need-emty
  transform(input) {}
};


test code
const babel = require('@babel/core');
const plugin = require('./babel-plugin-empty-functions-advanced.js');
const fs = require('fs');

// 测试代码
const testCode = `
// @need-emty
function test1() {
  console.log('test1');
  return 42;
}

// @need-emty
const test2 = () => {
  console.log('test2');
  return 'hello';
};

// 正常函数，不会被转换
function normal() {
  return 'normal';
}
`;

// 执行转换
const result = babel.transformSync(testCode, {
  plugins: [[plugin, { 
    marker: '@need-emty',
    logTransformed: true 
  }]],
  sourceMaps: true,
  comments: true
});

console.log('转换后的代码:');
console.log(result.code);

console.log('\nSourceMap:');
console.log(JSON.stringify(result.map, null, 2));

// 保存结果
fs.writeFileSync('output.js', result.code);
fs.writeFileSync('output.js.map', JSON.stringify(result.map, null, 2));
