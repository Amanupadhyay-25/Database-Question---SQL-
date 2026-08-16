## 6. How do you call joined table data from C# using Entity Framework/LINQ?

We can use LINQ to join two entities based on their related keys and select the required fields.

```csharp
var result = await (
    from e in _context.Employees
    join d in _context.Departments
    on e.DepartmentId equals d.Id
    select new
    {
        EmployeeName = e.Name,
        DepartmentName = d.DepartmentName
    }
).ToListAsync();
