### Câu 1:  SQL Injection tại Handler khi build rawquery hiện tại bên mình đang xử lý vấn đề này như nào sẽ dùng parameterized query với FromSQLInterpolate hay truyền dạng param Delare của FromSQLRaw, tại em đang thấy bên mình không dùng tới 2 thằng này để xử lý param, nên không rõ là mình có filler từ trong Middleware Pipeline không 
```C#
        if (request.SortColumnAndOrder.Any()) // =>>  Raw Query when order by multi column
        {
            var PageIndex = request.PageIndex <= 0 ? PagedResult<Domain.Entities.Product>.DefaultPageIndex : request.PageIndex;
            var PageSize = request.PageSize <= 0
                ? PagedResult<Domain.Entities.Product>.DefaultPageSize
                : request.PageSize > PagedResult<Domain.Entities.Product>.UpperPageSize
                ? PagedResult<Domain.Entities.Product>.UpperPageSize : request.PageSize;

            // ============================================
            var productsQuery = string.IsNullOrWhiteSpace(request.SearchTerm)
                ? @$"SELECT * FROM {nameof(Domain.Entities.Product)} ORDER BY "
                : @$"SELECT * FROM {nameof(Domain.Entities.Product)}
                        WHERE {nameof(Domain.Entities.Product.Name)} LIKE '%{request.SearchTerm}%'
                        OR {nameof(Domain.Entities.Product.Description)} LIKE '%{request.SearchTerm}%'
                        ORDER BY ";

            foreach (var item in request.SortColumnAndOrder)
                productsQuery += item.Value == SortOrder.Descending
                    ? $"{item.Key} DESC, "
                    : $"{item.Key} ASC, ";

            productsQuery = productsQuery.Remove(productsQuery.Length - 2);

            productsQuery += $" OFFSET {(PageIndex - 1) * PageSize} ROWS FETCH NEXT {PageSize} ROWS ONLY";

            var products = await _context.Products.FromSqlRaw(productsQuery)
                .ToListAsync(cancellationToken: cancellationToken);

            var totalCount = await _context.Products.CountAsync(cancellationToken);

            var productPagedResult = PagedResult<Domain.Entities.Product>.Create(products,
                PageIndex,
                PageSize,
                totalCount);

            var result = _mapper.Map<PagedResult<Response.ProductResponse>>(productPagedResult);

            return Result.Success(result);
        }
```
<img width="1165" height="644" alt="image" src="https://github.com/user-attachments/assets/f79490e5-609d-4b7d-9de8-40cebc138ee2" />

<img width="1599" height="899" alt="image" src="https://github.com/user-attachments/assets/78103f19-d91a-4cb2-86bb-7dace82a475a" />
 - Như ở API trên em có inject một câu sql vào trong serchTerm = 'nong';DELETE FROM Product WHERE Name LIKE '%nong%';SELECT * FROM Product WHERE Name LIKE '%nong'
và khi chạy thì nó không được lọc mã độc mà em inject ra ngoài.

Đây là bảng Product trước khi em chạy:
Id                                  |Name         |Price|Description                         |
------------------------------------+-------------+-----+------------------------------------+
B1553301-8D38-4B33-888F-33F1829D623C|string       | 1.00|string                              |
21E6D899-7F8E-4751-95B4-4B96452AE8E2|string Second| 1.00|a32f42c3-32e6-4bbb-949d-c3fd1cc97fc2|
7FA07885-ABDD-4C03-9988-50938883F91D|nong         | 1.00|123                                 |
CB29E4BA-A91A-421C-9DF4-7A7D67C3F610|string Second| 1.00|faa5b5a0-2527-47bd-a6f9-83e4582a5bbb|
FAA5B5A0-2527-47BD-A6F9-83E4582A5BBB|string       | 1.00|string                              |
C8EF432E-7420-46F8-BF8D-9256E7B61025|nong Second  | 1.00|7fa07885-abdd-4c03-9988-50938883f91d|
A32F42C3-32E6-4BBB-949D-C3FD1CC97FC2|string       | 1.00|string                              |
1A2D1B38-6C9F-4D0E-8DE2-C68B5EFF1864|string Second| 1.00|b1553301-8d38-4b33-888f-33f1829d623c|

Sau đó: 
Id                                  |Name         |Price|Description                         |
------------------------------------+-------------+-----+------------------------------------+
B1553301-8D38-4B33-888F-33F1829D623C|string       | 1.00|string                              |
21E6D899-7F8E-4751-95B4-4B96452AE8E2|string Second| 1.00|a32f42c3-32e6-4bbb-949d-c3fd1cc97fc2|
CB29E4BA-A91A-421C-9DF4-7A7D67C3F610|string Second| 1.00|faa5b5a0-2527-47bd-a6f9-83e4582a5bbb|
FAA5B5A0-2527-47BD-A6F9-83E4582A5BBB|string       | 1.00|string                              |
A32F42C3-32E6-4BBB-949D-C3FD1CC97FC2|string       | 1.00|string                              |
1A2D1B38-6C9F-4D0E-8DE2-C68B5EFF1864|string Second| 1.00|b1553301-8d38-4b33-888f-33f1829d623c|

### Câu 2: Tại sao lại dispose db từ trong repositories, ví khi DI DBContext sẽ được tạo ra 1 instance cho 1 http request các service trong request này đều sử dụng chung 1 instance vừa được tạo, và khi dispose từ trong repo thì các lớp khác cũng sẽ không dùng được instance đó nữa => sảy ra lỗi 
 ```C#
 
public class RepositoryBase<TEntity, TKey> : IRepositoryBase<TEntity, TKey>, IDisposable
        where TEntity : DomainEntity<TKey>
{
    ...
    public void Dispose()
        => _context?.Dispose();
    ...
}

 ```
 - Tuy em không thấy nó đang được sử dụng ở đâu nhưng em thắc mắc là muốn hỏi một case thực tế sử dụng nó thế nào ạ?

 ###Câu 3: SaveChange đang không truyền cancellationToken có phải mục đích là vẫn muốn giữ request vẫn chạy khi user cancel request đó hay không hay là có mục đích nào khác
 ```C#
 public class EFUnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;

    public EFUnitOfWork(ApplicationDbContext context)
        => _context = context;

    public async Task SaveChangesAsync(CancellationToken cancellationToken = default)
        => await _context.SaveChangesAsync();

    async ValueTask IAsyncDisposable.DisposeAsync()
        => await _context.DisposeAsync();
}
```

###Câu 4: Ngoài ra em cũng muốn hỏi lại dù trước anh có trả lời 1 lần là cần break rule kiến trúc Clean Architecture để phù hợp với dự án và vì dự án muốn build một sql động để dễ custom flex query nên cần inject được thằng dbcontext vào Handler để xử lý. Nhưng nếu như này em mà gặp một ví dụ thwucj tế là em mà đang sử dụng SQLServer mà muốn thay sang PostgreSQL(vì muốn sử dụng một dùng addon riêng của nó) thì khi mình thay đổi sang một loại DB khác thì mình sẽ phải maintain toàn bộ phần Application đã inject DBContext nữa phải không ạ
```C#
public sealed class GetProductsQueryHandler : IQueryHandler<Query.GetProductsQuery, PagedResult<Response.ProductResponse>>
{
    private readonly IRepositoryBase<Domain.Entities.Product, Guid> _productRepository;
    private readonly IMapper _mapper;
    private readonly ApplicationDbContext _context; //Inject trực tiếp DBContext từ chính Persistence vào Application.Handler

    public GetProductsQueryHandler(IRepositoryBase<Domain.Entities.Product, Guid> productRepository,
        ApplicationDbContext context,
        IMapper mapper)
    {
        _productRepository = productRepository;
        _mapper = mapper;
        _context = context;
    }
        ...
```

 <img width="371" height="227" alt="image" src="https://github.com/user-attachments/assets/a2863e49-8f6a-47d6-a61e-cdedae0d89a1" />

