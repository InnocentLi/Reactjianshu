#!/usr/bin/env python3
"""
generate_tests.py
自动解析 Java 源码并生成 JUnit5 单元测试（示例脚本）
Python 3.8+，依赖：javalang、jinja2
---------------------------------------------------
主要特性
1. 智能占位值：按参数类型给出默认值
2. 方法去重：同名重载自动加后缀，避免编译冲突
3. 多断言模板：Optional、异步 Callback、普通返回/void/异常
4. 外部模板：--template 指定 .j2 自定义模板
5. 交互/命令行两种选法：不加 --methods 则交互式编号选择
6. 兼容旧版 Jinja2：无 selectattr 嵌套、注册自定义 filter
---------------------------------------------------
Usage
  python generate_tests.py Foo.java
  python generate_tests.py Foo.java --methods add subtract -o FooTest.java
  python generate_tests.py Foo.java --template my.j2
"""

import argparse, os, re, sys
import javalang
from jinja2 import Environment, FileSystemLoader


# ---------- 占位值策略 -------------------------------------------------- #

def default_value(java_type: str | None) -> str:
    """根据参数类型返回示例值字符串"""
    if not java_type:
        return "null"
    t = java_type.lower()

    # 基本类型
    if t in {"byte", "short", "int", "char"}:
        return "0"
    if t in {"long"}:      # long 结尾带 L
        return "0L"
    if t in {"float", "double"}:
        return "0.0"
    if t.startswith("bool"):
        return "false"

    # 常见引用类型
    if "string" in t:
        return '"test"'
    if re.fullmatch(r"(list|arraylist|collection)", t, re.I):
        return "List.of()"
    if re.fullmatch(r"(map|hashmap)", t, re.I):
        return "Map.of()"

    # 默认
    return "null"


# ---------- 解析 Java 源码 ---------------------------------------------- #

def parse_methods(java_code: str):
    """返回 (类名, 方法列表)"""
    tree = javalang.parse.parse(java_code)
    cls = ""
    methods = []

    for node in tree.types:
        if isinstance(node, javalang.tree.ClassDeclaration):
            cls = node.name
            for m in node.methods:
                params = [{
                    "name": p.name,
                    "type": getattr(p.type, "name", None)
                } for p in m.parameters]

                methods.append({
                    "base_name": m.name,     # 原始方法名
                    "name": m.name,          # 渲染时可能会改
                    "params": params,
                    "return_type": getattr(m.return_type, "name", None),
                    "may_throw": "throw" in (m.documentation or "").lower()
                })
    # 同名重载自动加后缀
    return cls, unique_names(methods)


def unique_names(meths):
    counter = {}
    for m in meths:
        base = m["base_name"]
        counter[base] = counter.get(base, 0) + 1
        if counter[base] > 1:
            suffix = "_".join(p["type"] or "Obj" for p in m["params"]) or "void"
            suffix = re.sub(r'\W+', "_", suffix)  # 去掉非法字符
            m["name"] = f"{base}_{suffix}"
    return meths


# ---------- 判断是否异步（最后一个参数名/类型带 Callback） ---------------- #

def mark_async(methods):
    for m in methods:
        m["is_async"] = (
            m["params"] and
            re.search(r"callback$", m["params"][-1]["type"] or "", re.I)
        )
    return methods


# ---------- 需要 import List/Map ? ------------------------------------- #

def need_imports(methods):
    types = {p["type"] or "" for m in methods for p in m["params"]}
    need_list = any(re.fullmatch(r"(list|arraylist|collection)", t, re.I) for t in types)
    need_map  = any(re.fullmatch(r"(map|hashmap)", t, re.I) for t in types)
    return need_list, need_map


# ---------- Jinja2 模板 ------------------------------------------------- #

DEFAULT_TEMPLATE = r"""
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
{% if needList %}import java.util.List;{% endif %}
{% if needMap  %}import java.util.Map;{% endif %}
import java.util.concurrent.*;

public class {{ cls }}Test {
{% for m in methods %}
    @Test
    void {{ m.name }}() {
        // arrange
        {{ cls }} obj = new {{ cls }}();
{% if m.is_async %}
        java.util.concurrent.CompletableFuture<Object> future = new java.util.concurrent.CompletableFuture<>();
{% endif %}

        // act
{% if m.is_async %}
        obj.{{ m.base_name }}({{ m.params|join_param }}, future::complete);
{% elif m.return_type %}
        var result = obj.{{ m.base_name }}({{ m.params|join_param }});
{% else %}
        obj.{{ m.base_name }}({{ m.params|join_param }});
{% endif %}

        // assert
{% if m.may_throw %}
        // TODO: assertThrows(...)
{% elif m.is_async %}
        assertEquals(/* expected */ 0, future.get(1, java.util.concurrent.TimeUnit.SECONDS));
{% elif m.return_type == None %}
        // TODO: verify side-effects
{% elif m.return_type == "OptionalDouble" %}
        assertTrue(result.isPresent());
        // assertEquals(expected, result.getAsDouble());
{% else %}
        assertEquals(/* expected */ 0, result);
{% endif %}
    }
{% endfor %}
}

"""


def build_template(path: str | None):
    env = Environment(
        loader=FileSystemLoader(os.path.dirname(path) or ".") if path else None,
        trim_blocks=True,
        lstrip_blocks=True,
    )

    # join_param：把参数占位值按逗号连接
    def join_param(params):
        return ", ".join(default_value(p["type"]) for p in params)

    env.filters["join_param"] = join_param

    if path:
        return env.get_template(os.path.basename(path))
    return env.from_string(DEFAULT_TEMPLATE)


# ---------- 渲染 -------------------------------------------------------- #

def render(cls, methods, template):
    need_list, need_map = need_imports(methods)
    return template.render(
        cls=cls,
        methods=methods,
        needList=need_list,
        needMap=need_map
    )


# ---------- CLI 主入口 -------------------------------------------------- #

def main():
    ap = argparse.ArgumentParser(description="Generate JUnit5 tests from Java source")
    ap.add_argument("java_file", help="Java 源码文件路径")
    ap.add_argument("-o", "--output", help="输出测试文件名 (.java)")
    ap.add_argument("--methods", nargs="+", help="只为列出的方法生成测试")
    ap.add_argument("--template", help="自定义 Jinja2 模板文件 (.j2)")
    args = ap.parse_args()

    if not os.path.exists(args.java_file):
        sys.exit("❌ Java 文件不存在")

    src = open(args.java_file, encoding="utf-8").read()
    cls, all_methods = parse_methods(src)

    # 选方法：命令行或交互
    if args.methods:
        sel = set(args.methods)
        methods = [m for m in all_methods if m["base_name"] in sel]
    else:
        print("\n可用方法：")
        for i, m in enumerate(all_methods):
            sig = ", ".join(f"{p['type']} {p['name']}" for p in m["params"])
            print(f"  {i}: {m['base_name']}({sig})")
        idxs = input("输入编号（逗号分隔，直接回车=全部）：").strip()
        if idxs:
            choose = {int(i.strip()) for i in idxs.split(",") if i.strip().isdigit()}
            methods = [all_methods[i] for i in choose]
        else:
            methods = all_methods

    methods = mark_async(methods)
    template = build_template(args.template)
    code = render(cls, methods, template)

    out_file = args.output or f"{cls}Test.java"
    with open(out_file, "w", encoding="utf-8") as f:
        f.write(code)
    print(f"✅ 测试代码已生成：{out_file}")


if __name__ == "__main__":
    main()
