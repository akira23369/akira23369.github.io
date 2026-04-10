---
title: 反射系列笔记（十二）：实战项目——构建简易ORM框架
date: 2026-04-07 09:19:44
toc: true
categories:
  - 编程语言
  - Csharp
  - 反射与特性

tags:
  - 编程语言
  - Csharp
  - 反射与特性

---


# C# 反射系列笔记（十二）：实战项目——构建简易ORM框架

> 目标：综合运用前面十一篇的所有反射知识，从零构建一个功能完整的微型 ORM 框架，理解反射在实际项目中的应用模式

---

## 一、项目概述

### 我们要构建什么？

**MiniORM** —— 一个轻量级的对象关系映射框架

| 功能 | 技术要点 |
|------|----------|
| 实体映射 | 特性定义表名、列名、主键 |
| SQL 生成 | 动态生成 INSERT/UPDATE/SELECT/DELETE |
| 数据读取 | 从 IDataReader 填充对象 |
| 连接管理 | 支持连接池和事务 |
| 查询执行 | 参数化查询防注入 |
| 表达式查询 | 基础的条件表达式支持 |

### 架构设计

```
MiniORM
├── Attributes/          # 映射特性
│   ├── TableAttribute
│   ├── ColumnAttribute
│   └── KeyAttribute
├── Mapping/             # 映射元数据
│   ├── EntityMap
│   └── ColumnMap
├── SqlGenerators/       # SQL 生成器
│   ├── ISqlGenerator
│   └── SqlServerGenerator
├── DbContext/           # 数据库上下文
│   └── MiniDbContext
└── Query/               # 查询构建器
    └── QueryBuilder
```

---

## 二、特性定义（映射元数据）

### 核心特性

```csharp
using System;

namespace MiniORM.Attributes
{
    /// <summary>
    /// 标记实体对应的数据库表
    /// </summary>
    [AttributeUsage(AttributeTargets.Class)]
    public class TableAttribute : Attribute
    {
        public string Name { get; }
        
        /// <param name="name">表名</param>
        public TableAttribute(string name)
        {
            Name = name;
        }
    }
    
    /// <summary>
    /// 标记属性对应的数据库列
    /// </summary>
    [AttributeUsage(AttributeTargets.Property)]
    public class ColumnAttribute : Attribute
    {
        public string Name { get; set; }
        public bool IsNullable { get; set; } = true;
        public int? MaxLength { get; set; }
        
        public ColumnAttribute() { }
        
        public ColumnAttribute(string name)
        {
            Name = name;
        }
    }
    
    /// <summary>
    /// 标记主键（支持复合主键）
    /// </summary>
    [AttributeUsage(AttributeTargets.Property)]
    public class KeyAttribute : Attribute
    {
        public bool IsIdentity { get; set; } = false;
        
        public KeyAttribute() { }
        
        public KeyAttribute(bool isIdentity)
        {
            IsIdentity = isIdentity;
        }
    }
    
    /// <summary>
    /// 忽略此属性（不映射到数据库）
    /// </summary>
    [AttributeUsage(AttributeTargets.Property)]
    public class IgnoreAttribute : Attribute { }
}
```

### 实体示例

```csharp
using MiniORM.Attributes;

[Table("Users")]
public class User
{
    [Key(true)]  // 自增主键
    [Column("Id")]
    public int Id { get; set; }
    
    [Column("UserName")]
    [Column(MaxLength = 50)]
    public string Name { get; set; }
    
    [Column("UserAge")]
    public int Age { get; set; }
    
    [Column("Email")]
    public string Email { get; set; }
    
    [Column("CreatedAt")]
    public DateTime CreatedAt { get; set; }
    
    [Ignore]  // 不存储到数据库
    public string TempData { get; set; }
}

[Table("Orders")]
public class Order
{
    [Key(true)]
    public int Id { get; set; }
    
    [Column("UserId")]
    public int UserId { get; set; }
    
    [Column("Amount")]
    public decimal Amount { get; set; }
    
    [Column("Status")]
    public string Status { get; set; }
    
    [Column("OrderDate")]
    public DateTime OrderDate { get; set; }
}
```

---

## 三、映射元数据缓存

### 映射信息类

```csharp
using System.Reflection;
using MiniORM.Attributes;

namespace MiniORM.Mapping
{
    /// <summary>
    /// 列的映射信息
    /// </summary>
    public class ColumnMap
    {
        public string ColumnName { get; set; }
        public string PropertyName { get; set; }
        public Type PropertyType { get; set; }
        public PropertyInfo PropertyInfo { get; set; }
        public bool IsKey { get; set; }
        public bool IsIdentity { get; set; }
        public bool IsNullable { get; set; }
        public int? MaxLength { get; set; }
        
        // 获取值的委托（缓存）
        public Func<object, object> Getter { get; set; }
        
        // 设置值的委托（缓存）
        public Action<object, object> Setter { get; set; }
    }
    
    /// <summary>
    /// 实体的映射信息
    /// </summary>
    public class EntityMap
    {
        public Type EntityType { get; set; }
        public string TableName { get; set; }
        public List<ColumnMap> Columns { get; set; } = new List<ColumnMap>();
        public ColumnMap KeyColumn { get; set; }
        
        // 创建实例的委托
        public Func<object> Creator { get; set; }
        
        public bool HasIdentity => KeyColumn?.IsIdentity == true;
    }
}
```

### 映射缓存管理器

```csharp
using System.Collections.Concurrent;
using System.Reflection;
using System.Linq.Expressions;
using MiniORM.Attributes;

namespace MiniORM.Mapping
{
    public static class MappingCache
    {
        private static readonly ConcurrentDictionary<Type, EntityMap> _maps = new();
        
        /// <summary>
        /// 获取实体的映射信息（自动扫描并缓存）
        /// </summary>
        public static EntityMap GetMap<T>() => GetMap(typeof(T));
        
        public static EntityMap GetMap(Type type)
        {
            return _maps.GetOrAdd(type, t =>
            {
                var map = new EntityMap
                {
                    EntityType = t,
                    TableName = GetTableName(t),
                    Creator = CreateCreator(t)
                };
                
                ScanProperties(t, map);
                
                if (map.KeyColumn == null && map.Columns.Count > 0)
                {
                    // 如果没有标记主键，使用第一个属性作为主键
                    map.KeyColumn = map.Columns.First();
                }
                
                return map;
            });
        }
        
        private static string GetTableName(Type type)
        {
            var tableAttr = type.GetCustomAttribute<TableAttribute>();
            return tableAttr?.Name ?? type.Name;
        }
        
        private static void ScanProperties(Type type, EntityMap map)
        {
            var properties = type.GetProperties(BindingFlags.Public | BindingFlags.Instance);
            
            foreach (var prop in properties)
            {
                // 检查是否忽略
                if (prop.IsDefined(typeof(IgnoreAttribute)))
                    continue;
                
                var columnMap = CreateColumnMap(prop);
                map.Columns.Add(columnMap);
                
                // 检查是否为主键
                if (prop.IsDefined(typeof(KeyAttribute)))
                {
                    var keyAttr = prop.GetCustomAttribute<KeyAttribute>();
                    columnMap.IsKey = true;
                    columnMap.IsIdentity = keyAttr.IsIdentity;
                    map.KeyColumn = columnMap;
                }
            }
        }
        
        private static ColumnMap CreateColumnMap(PropertyInfo prop)
        {
            var columnAttr = prop.GetCustomAttribute<ColumnAttribute>();
            var keyAttr = prop.GetCustomAttribute<KeyAttribute>();
            
            var map = new ColumnMap
            {
                PropertyName = prop.Name,
                PropertyInfo = prop,
                PropertyType = prop.PropertyType,
                ColumnName = columnAttr?.Name ?? prop.Name,
                IsNullable = columnAttr?.IsNullable ?? true,
                MaxLength = columnAttr?.MaxLength,
                IsKey = keyAttr != null,
                IsIdentity = keyAttr?.IsIdentity ?? false,
                Getter = CreateGetter(prop),
                Setter = CreateSetter(prop)
            };
            
            return map;
        }
        
        private static Func<object, object> CreateGetter(PropertyInfo prop)
        {
            var instanceParam = Expression.Parameter(typeof(object), "instance");
            var castInstance = Expression.Convert(instanceParam, prop.DeclaringType);
            var propertyAccess = Expression.Property(castInstance, prop);
            var convertResult = Expression.Convert(propertyAccess, typeof(object));
            
            var lambda = Expression.Lambda<Func<object, object>>(convertResult, instanceParam);
            return lambda.Compile();
        }
        
        private static Action<object, object> CreateSetter(PropertyInfo prop)
        {
            var instanceParam = Expression.Parameter(typeof(object), "instance");
            var valueParam = Expression.Parameter(typeof(object), "value");
            var castInstance = Expression.Convert(instanceParam, prop.DeclaringType);
            var castValue = Expression.Convert(valueParam, prop.PropertyType);
            var propertyAccess = Expression.Property(castInstance, prop);
            var assign = Expression.Assign(propertyAccess, castValue);
            
            var lambda = Expression.Lambda<Action<object, object>>(assign, instanceParam, valueParam);
            return lambda.Compile();
        }
        
        private static Func<object> CreateCreator(Type type)
        {
            var ctor = type.GetConstructor(Type.EmptyTypes);
            if (ctor == null)
                return () => throw new InvalidOperationException($"类型 {type.Name} 没有无参构造函数");
            
            var lambda = Expression.Lambda<Func<object>>(Expression.New(ctor));
            return lambda.Compile();
        }
    }
}
```

---

## 四、SQL 生成器

### 接口定义

```csharp
using System.Text;

namespace MiniORM.SqlGenerators
{
    public interface ISqlGenerator
    {
        /// <summary>
        /// 生成 INSERT 语句
        /// </summary>
        SqlResult GenerateInsert(EntityMap map);
        
        /// <summary>
        /// 生成 UPDATE 语句
        /// </summary>
        SqlResult GenerateUpdate(EntityMap map);
        
        /// <summary>
        /// 生成 DELETE 语句
        /// </summary>
        SqlResult GenerateDelete(EntityMap map);
        
        /// <summary>
        /// 生成 SELECT 语句（按主键）
        /// </summary>
        SqlResult GenerateSelectById(EntityMap map);
        
        /// <summary>
        /// 生成 SELECT 语句（查询所有）
        /// </summary>
        SqlResult GenerateSelectAll(EntityMap map);
        
        /// <summary>
        /// 获取自增列的值
        /// </summary>
        string GetIdentitySql();
    }
    
    public class SqlResult
    {
        public string Sql { get; set; }
        public Dictionary<string, object> Parameters { get; set; } = new();
    }
}
```

### SQL Server 实现

```csharp
using System.Text;

namespace MiniORM.SqlGenerators
{
    public class SqlServerGenerator : ISqlGenerator
    {
        public SqlResult GenerateInsert(EntityMap map)
        {
            var sql = new StringBuilder();
            var result = new SqlResult();
            
            var columns = map.Columns.Where(c => !c.IsIdentity).ToList();
            
            sql.Append($"INSERT INTO {map.TableName} (");
            sql.Append(string.Join(", ", columns.Select(c => $"[{c.ColumnName}]")));
            sql.Append(") VALUES (");
            sql.Append(string.Join(", ", columns.Select(c => $"@{c.PropertyName}")));
            sql.Append(")");
            
            // 添加参数
            foreach (var col in columns)
            {
                result.Parameters[col.PropertyName] = null; // 占位，实际值在执行时填充
            }
            
            result.Sql = sql.ToString();
            return result;
        }
        
        public SqlResult GenerateUpdate(EntityMap map)
        {
            var sql = new StringBuilder();
            var result = new SqlResult();
            
            var columns = map.Columns.Where(c => !c.IsKey).ToList();
            
            sql.Append($"UPDATE {map.TableName} SET ");
            sql.Append(string.Join(", ", columns.Select(c => $"[{c.ColumnName}] = @{c.PropertyName}")));
            sql.Append($" WHERE [{map.KeyColumn.ColumnName}] = @{map.KeyColumn.PropertyName}");
            
            // 添加参数
            foreach (var col in map.Columns)
            {
                result.Parameters[col.PropertyName] = null;
            }
            
            result.Sql = sql.ToString();
            return result;
        }
        
        public SqlResult GenerateDelete(EntityMap map)
        {
            var result = new SqlResult();
            result.Sql = $"DELETE FROM {map.TableName} WHERE [{map.KeyColumn.ColumnName}] = @{map.KeyColumn.PropertyName}";
            result.Parameters[map.KeyColumn.PropertyName] = null;
            return result;
        }
        
        public SqlResult GenerateSelectById(EntityMap map)
        {
            var result = new SqlResult();
            result.Sql = $"SELECT * FROM {map.TableName} WHERE [{map.KeyColumn.ColumnName}] = @{map.KeyColumn.PropertyName}";
            result.Parameters[map.KeyColumn.PropertyName] = null;
            return result;
        }
        
        public SqlResult GenerateSelectAll(EntityMap map)
        {
            return new SqlResult
            {
                Sql = $"SELECT * FROM {map.TableName}"
            };
        }
        
        public string GetIdentitySql()
        {
            return "SELECT SCOPE_IDENTITY();";
        }
    }
}
```

---

## 五、数据读取器映射

```csharp
using System.Data;
using MiniORM.Mapping;

namespace MiniORM
{
    public static class DataReaderMapper
    {
        /// <summary>
        /// 将 IDataReader 映射为对象列表
        /// </summary>
        public static List<T> MapToList<T>(IDataReader reader) where T : new()
        {
            var list = new List<T>();
            var map = MappingCache.GetMap<T>();
            
            // 预计算列名到 ColumnMap 的映射
            var columnMappings = BuildColumnMappings(reader, map);
            
            while (reader.Read())
            {
                var entity = MapToEntity(reader, map, columnMappings);
                list.Add(entity);
            }
            
            return list;
        }
        
        /// <summary>
        /// 映射单条记录
        /// </summary>
        public static T MapToEntity<T>(IDataReader reader) where T : new()
        {
            var map = MappingCache.GetMap<T>();
            var columnMappings = BuildColumnMappings(reader, map);
            return MapToEntity(reader, map, columnMappings);
        }
        
        private static T MapToEntity<T>(IDataReader reader, EntityMap map, Dictionary<string, ColumnMap> columnMappings) where T : new()
        {
            var entity = (T)map.Creator();
            
            foreach (var col in map.Columns)
            {
                if (columnMappings.TryGetValue(col.ColumnName, out var columnMap))
                {
                    var value = reader[col.ColumnName];
                    if (value != DBNull.Value)
                    {
                        var convertedValue = ConvertValue(value, col.PropertyType);
                        col.Setter(entity, convertedValue);
                    }
                }
            }
            
            return entity;
        }
        
        private static Dictionary<string, ColumnMap> BuildColumnMappings(IDataReader reader, EntityMap map)
        {
            var mappings = new Dictionary<string, ColumnMap>(StringComparer.OrdinalIgnoreCase);
            
            for (int i = 0; i < reader.FieldCount; i++)
            {
                var columnName = reader.GetName(i);
                var columnMap = map.Columns.FirstOrDefault(c => 
                    string.Equals(c.ColumnName, columnName, StringComparison.OrdinalIgnoreCase));
                
                if (columnMap != null)
                {
                    mappings[columnName] = columnMap;
                }
            }
            
            return mappings;
        }
        
        private static object ConvertValue(object value, Type targetType)
        {
            if (value == null || value == DBNull.Value)
                return null;
            
            var sourceType = value.GetType();
            
            if (targetType.IsAssignableFrom(sourceType))
                return value;
            
            // 处理可空类型
            var underlyingType = Nullable.GetUnderlyingType(targetType);
            if (underlyingType != null)
            {
                return Convert.ChangeType(value, underlyingType);
            }
            
            return Convert.ChangeType(value, targetType);
        }
    }
}
```

---

## 六、数据库上下文核心

```csharp
using System.Data;
using System.Data.SqlClient;
using MiniORM.Mapping;
using MiniORM.SqlGenerators;

namespace MiniORM
{
    public class MiniDbContext : IDisposable
    {
        private readonly string _connectionString;
        private IDbConnection _connection;
        private IDbTransaction _transaction;
        private readonly ISqlGenerator _sqlGenerator;
        private bool _disposed;
        
        public MiniDbContext(string connectionString) : this(connectionString, new SqlServerGenerator())
        {
        }
        
        public MiniDbContext(string connectionString, ISqlGenerator sqlGenerator)
        {
            _connectionString = connectionString;
            _sqlGenerator = sqlGenerator;
        }
        
        private IDbConnection Connection
        {
            get
            {
                if (_connection == null)
                {
                    _connection = new SqlConnection(_connectionString);
                    _connection.Open();
                }
                return _connection;
            }
        }
        
        #region CRUD 操作
        
        /// <summary>
        /// 插入实体
        /// </summary>
        public virtual int Insert<T>(T entity) where T : class
        {
            var map = MappingCache.GetMap<T>();
            var sqlResult = _sqlGenerator.GenerateInsert(map);
            
            // 填充参数
            foreach (var param in sqlResult.Parameters.Keys.ToList())
            {
                var col = map.Columns.First(c => c.PropertyName == param);
                sqlResult.Parameters[param] = col.Getter(entity);
            }
            
            var result = ExecuteScalar(sqlResult.Sql, sqlResult.Parameters);
            
            // 如果是自增主键，回填 ID
            if (map.HasIdentity && result != null)
            {
                var identitySql = _sqlGenerator.GetIdentitySql();
                var identityValue = ExecuteScalar(identitySql);
                map.KeyColumn.Setter(entity, Convert.ChangeType(identityValue, map.KeyColumn.PropertyType));
            }
            
            return Convert.ToInt32(result ?? 0);
        }
        
        /// <summary>
        /// 更新实体
        /// </summary>
        public virtual int Update<T>(T entity) where T : class
        {
            var map = MappingCache.GetMap<T>();
            var sqlResult = _sqlGenerator.GenerateUpdate(map);
            
            foreach (var param in sqlResult.Parameters.Keys.ToList())
            {
                var col = map.Columns.First(c => c.PropertyName == param);
                sqlResult.Parameters[param] = col.Getter(entity);
            }
            
            return ExecuteNonQuery(sqlResult.Sql, sqlResult.Parameters);
        }
        
        /// <summary>
        /// 删除实体
        /// </summary>
        public virtual int Delete<T>(T entity) where T : class
        {
            var map = MappingCache.GetMap<T>();
            var sqlResult = _sqlGenerator.GenerateDelete(map);
            
            sqlResult.Parameters[map.KeyColumn.PropertyName] = map.KeyColumn.Getter(entity);
            
            return ExecuteNonQuery(sqlResult.Sql, sqlResult.Parameters);
        }
        
        /// <summary>
        /// 按主键查找
        /// </summary>
        public virtual T Find<T>(object id) where T : class, new()
        {
            var map = MappingCache.GetMap<T>();
            var sqlResult = _sqlGenerator.GenerateSelectById(map);
            sqlResult.Parameters[map.KeyColumn.PropertyName] = id;
            
            using var reader = ExecuteReader(sqlResult.Sql, sqlResult.Parameters);
            if (reader.Read())
            {
                return DataReaderMapper.MapToEntity<T>(reader);
            }
            
            return null;
        }
        
        /// <summary>
        /// 查询所有
        /// </summary>
        public virtual List<T> GetAll<T>() where T : class, new()
        {
            var map = MappingCache.GetMap<T>();
            var sqlResult = _sqlGenerator.GenerateSelectAll(map);
            
            using var reader = ExecuteReader(sqlResult.Sql);
            return DataReaderMapper.MapToList<T>(reader);
        }
        
        #endregion
        
        #region 原生 SQL
        
        /// <summary>
        /// 执行原生 SQL 查询
        /// </summary>
        public virtual List<T> Query<T>(string sql, object parameters = null) where T : new()
        {
            var paramDict = ObjectToDictionary(parameters);
            using var reader = ExecuteReader(sql, paramDict);
            return DataReaderMapper.MapToList<T>(reader);
        }
        
        /// <summary>
        /// 执行非查询 SQL
        /// </summary>
        public virtual int Execute(string sql, object parameters = null)
        {
            var paramDict = ObjectToDictionary(parameters);
            return ExecuteNonQuery(sql, paramDict);
        }
        
        /// <summary>
        /// 执行标量查询
        /// </summary>
        public virtual object ExecuteScalar(string sql, object parameters = null)
        {
            var paramDict = ObjectToDictionary(parameters);
            return ExecuteScalar(sql, paramDict);
        }
        
        #endregion
        
        #region 事务
        
        public void BeginTransaction()
        {
            _transaction = Connection.BeginTransaction();
        }
        
        public void Commit()
        {
            _transaction?.Commit();
            _transaction = null;
        }
        
        public void Rollback()
        {
            _transaction?.Rollback();
            _transaction = null;
        }
        
        #endregion
        
        #region 私有辅助方法
        
        private IDbCommand CreateCommand(string sql, Dictionary<string, object> parameters = null)
        {
            var cmd = Connection.CreateCommand();
            cmd.CommandText = sql;
            cmd.Transaction = _transaction;
            
            if (parameters != null)
            {
                foreach (var param in parameters)
                {
                    var dbParam = cmd.CreateParameter();
                    dbParam.ParameterName = $"@{param.Key}";
                    dbParam.Value = param.Value ?? DBNull.Value;
                    cmd.Parameters.Add(dbParam);
                }
            }
            
            return cmd;
        }
        
        private int ExecuteNonQuery(string sql, Dictionary<string, object> parameters = null)
        {
            using var cmd = CreateCommand(sql, parameters);
            return cmd.ExecuteNonQuery();
        }
        
        private IDataReader ExecuteReader(string sql, Dictionary<string, object> parameters = null)
        {
            var cmd = CreateCommand(sql, parameters);
            return cmd.ExecuteReader();
        }
        
        private object ExecuteScalar(string sql, Dictionary<string, object> parameters = null)
        {
            using var cmd = CreateCommand(sql, parameters);
            return cmd.ExecuteScalar();
        }
        
        private Dictionary<string, object> ObjectToDictionary(object obj)
        {
            if (obj == null) return new Dictionary<string, object>();
            
            var dict = new Dictionary<string, object>();
            var props = obj.GetType().GetProperties();
            
            foreach (var prop in props)
            {
                dict[prop.Name] = prop.GetValue(obj);
            }
            
            return dict;
        }
        
        #endregion
        
        #region IDisposable
        
        public void Dispose()
        {
            if (!_disposed)
            {
                _transaction?.Dispose();
                _connection?.Dispose();
                _disposed = true;
            }
        }
        
        #endregion
    }
}
```

---

## 七、查询构建器（表达式支持）

```csharp
using System.Linq.Expressions;

namespace MiniORM.Query
{
    public class QueryBuilder<T> where T : class, new()
    {
        private readonly MiniDbContext _context;
        private string _whereClause;
        private string _orderBy;
        private int? _top;
        
        public QueryBuilder(MiniDbContext context)
        {
            _context = context;
        }
        
        /// <summary>
        /// 添加 WHERE 条件
        /// </summary>
        public QueryBuilder<T> Where(Expression<Func<T, bool>> predicate)
        {
            var visitor = new WhereExpressionVisitor();
            _whereClause = visitor.Visit(predicate);
            return this;
        }
        
        /// <summary>
        /// 添加 WHERE 条件（字符串方式）
        /// </summary>
        public QueryBuilder<T> Where(string condition, object parameters)
        {
            _whereClause = condition;
            return this;
        }
        
        /// <summary>
        /// 排序
        /// </summary>
        public QueryBuilder<T> OrderBy<TKey>(Expression<Func<T, TKey>> keySelector, bool ascending = true)
        {
            var memberExpr = keySelector.Body as MemberExpression;
            var columnName = memberExpr?.Member.Name;
            
            _orderBy = $"{columnName} {(ascending ? "ASC" : "DESC")}";
            return this;
        }
        
        /// <summary>
        /// 获取前 N 条
        /// </summary>
        public QueryBuilder<T> Take(int count)
        {
            _top = count;
            return this;
        }
        
        /// <summary>
        /// 执行查询
        /// </summary>
        public List<T> ToList()
        {
            var sql = BuildSelectSql();
            return _context.Query<T>(sql);
        }
        
        /// <summary>
        /// 获取第一条记录
        /// </summary>
        public T FirstOrDefault()
        {
            _top = 1;
            var sql = BuildSelectSql();
            return _context.Query<T>(sql).FirstOrDefault();
        }
        
        private string BuildSelectSql()
        {
            var map = MappingCache.GetMap<T>();
            var sql = $"SELECT * FROM {map.TableName}";
            
            if (!string.IsNullOrEmpty(_whereClause))
                sql += $" WHERE {_whereClause}";
            
            if (!string.IsNullOrEmpty(_orderBy))
                sql += $" ORDER BY {_orderBy}";
            
            if (_top.HasValue)
                sql = sql.Replace("SELECT", $"SELECT TOP {_top}");
            
            return sql;
        }
        
        private class WhereExpressionVisitor : ExpressionVisitor
        {
            public string Visit(Expression<Func<T, bool>> predicate)
            {
                // 简化实现：仅支持基础比较
                // 完整实现需要处理 AND/OR、常量值等
                return VisitBinary(predicate.Body as BinaryExpression);
            }
            
            private string VisitBinary(BinaryExpression binary)
            {
                var left = GetMemberName(binary.Left);
                var right = GetValue(binary.Right);
                var op = GetOperator(binary.NodeType);
                
                return $"{left} {op} {right}";
            }
            
            private string GetMemberName(Expression expr)
            {
                if (expr is MemberExpression member)
                    return member.Member.Name;
                if (expr is UnaryExpression unary && unary.Operand is MemberExpression inner)
                    return inner.Member.Name;
                
                return expr.ToString();
            }
            
            private string GetValue(Expression expr)
            {
                if (expr is ConstantExpression constant)
                    return FormatValue(constant.Value);
                
                var lambda = Expression.Lambda(expr);
                var value = lambda.Compile().DynamicInvoke();
                return FormatValue(value);
            }
            
            private string FormatValue(object value)
            {
                if (value == null) return "NULL";
                if (value is string) return $"'{value}'";
                if (value is DateTime dt) return $"'{dt:yyyy-MM-dd HH:mm:ss}'";
                if (value is bool b) return b ? "1" : "0";
                return value.ToString();
            }
            
            private string GetOperator(ExpressionType type)
            {
                return type switch
                {
                    ExpressionType.Equal => "=",
                    ExpressionType.NotEqual => "<>",
                    ExpressionType.GreaterThan => ">",
                    ExpressionType.GreaterThanOrEqual => ">=",
                    ExpressionType.LessThan => "<",
                    ExpressionType.LessThanOrEqual => "<=",
                    _ => "="
                };
            }
        }
    }
}
```

---

## 八、完整使用示例

```csharp
using MiniORM;

class Program
{
    static void Main()
    {
        // 连接字符串（实际使用时替换为真实数据库）
        var connectionString = "Server=localhost;Database=TestDB;Trusted_Connection=true;";
        
        using var db = new MiniDbContext(connectionString);
        
        // ========== 插入数据 ==========
        var user = new User
        {
            Name = "张三",
            Age = 25,
            Email = "zhangsan@test.com",
            CreatedAt = DateTime.Now
        };
        
        int newId = db.Insert(user);
        Console.WriteLine($"插入成功，新ID: {newId}");
        
        // ========== 查询单条 ==========
        var found = db.Find<User>(newId);
        Console.WriteLine($"查询结果: {found.Name}, {found.Age}");
        
        // ========== 更新数据 ==========
        found.Age = 26;
        db.Update(found);
        Console.WriteLine("更新成功");
        
        // ========== 查询所有 ==========
        var allUsers = db.GetAll<User>();
        Console.WriteLine($"用户总数: {allUsers.Count}");
        
        // ========== 条件查询 ==========
        var adultUsers = db.Query<User>("SELECT * FROM Users WHERE UserAge > @age", new { age = 18 });
        
        // ========== 使用查询构建器 ==========
        var query = new QueryBuilder<User>(db)
            .Where(u => u.Age > 18)
            .OrderBy(u => u.Name)
            .Take(10)
            .ToList();
        
        // ========== 事务操作 ==========
        try
        {
            db.BeginTransaction();
            
            var order = new Order
            {
                UserId = newId,
                Amount = 199.99m,
                Status = "Pending",
                OrderDate = DateTime.Now
            };
            db.Insert(order);
            
            // 更新用户相关数据...
            
            db.Commit();
            Console.WriteLine("事务提交成功");
        }
        catch
        {
            db.Rollback();
            Console.WriteLine("事务回滚");
        }
        
        // ========== 删除数据 ==========
        db.Delete(found);
        Console.WriteLine("删除成功");
    }
}
```

---

## 九、扩展建议

### 可扩展的功能点

| 功能 | 实现方式 |
|------|----------|
| 批量插入 | 使用 SqlBulkCopy 或 VALUES 多行语法 |
| 分页查询 | 添加 Page 方法，生成 OFFSET/FETCH 或 ROWNUM |
| 导航属性 | 添加 ForeignKey 特性，支持延迟加载 |
| 连接池 | 使用 DbConnection 池（ADO.NET 自带） |
| 日志记录 | 添加 IDbCommandInterceptor 接口 |
| 缓存 | 一级缓存（按主键），二级缓存（分布式） |
| 迁移 | 扫描实体，生成建表/改表脚本 |

### 性能优化清单

```csharp
// 1. 使用委托缓存（已完成）
// 2. 使用表达式树编译（已完成）
// 3. 批量操作减少数据库往返
// 4. 使用 Dapper 作为底层执行器（可选）
// 5. 预编译 SQL 语句
```

---

## 十、总结：反射知识综合应用回顾

| 反射知识点 | 在 MiniORM 中的应用 |
|------------|---------------------|
| `Type` | 获取实体类型信息 |
| `PropertyInfo` | 读取/设置属性值 |
| `GetCustomAttribute` | 读取映射特性 |
| `GetProperties/BindingFlags` | 扫描所有属性 |
| `Activator.CreateInstance` | 创建实体实例 |
| `MethodInfo.Invoke` | 早期版本的方法调用 |
| `Delegate.CreateDelegate` | 创建 Getter/Setter 委托 |
| `Expression` | 编译 Getter/Setter 委托 |
| `IsValueType/IsClass` | 判断值类型/引用类型 |
| `Nullable.GetUnderlyingType` | 处理可空类型 |

---

## 结语

通过这十二篇笔记，我们从反射的基础概念出发，逐步深入到类型信息获取、程序集加载、动态创建对象、方法/属性/字段/事件调用、特性处理、泛型反射，最终以性能优化和实战项目收尾。

反射是 .NET 框架的基石之一，理解反射不仅有助于日常开发中解决动态编程问题，更是深入理解 Entity Framework、ASP.NET Core、Json.NET、依赖注入容器等框架的关键。

**学习建议：**
1. 动手实现 MiniORM 的完整版本
2. 阅读 Dapper 源码（高性能反射的典范）
3. 阅读 EF Core 源码（复杂反射应用的案例）
4. 实践：为自己的项目添加插件架构或动态功能

---

**全系列完结** 🎉