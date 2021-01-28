## Proper WebAPI Pattern

Steps to scafold an efficient and organised way for WebAPIs

### Express Global Exception Handler
Use the global exception handler to get a standard response object and taking care of 500 Internal server errors. Register on Startup.cs like this

```csharp
app.UseGlobalExceptionHandler();
```

### Endpoints With BadRequest Validator
Endpoints should implement request validators and give appropriate info to the developer using the API. A good API need to do it

```csharp
        [HttpPost]
        [Route("LogIn")]
        public IActionResult LogIn([FromBody] LoginParameters data)
        {
            if (UsersValidator.ValidateLogin(data).IsSuccess)
            {
                var response = authService.Login(data.Username, data.Password);
                return (response.IsSuccess) ? Ok(response) : BadRequest(response);
            }
            else
            {
                return BadRequest(UsersValidator.ValidateLogin(data));
            }
        }
```
Validators can be like this

```csharp
    public static class UsersValidator
    {
        private static List<string> Errors = new List<string>();

        public static Response ValidateLogin(LoginParameters data)
        {
            Errors.Clear();
            try
            {
                if (data.Username == null || data.Username.Length < 1)
                {
                    Errors.Add("Username should have more than 1 charactor");
                }
                if (data.Password == null || data.Password.Length < 1)
                {
                    Errors.Add("Password should have more than 1 charactor");
                }
                return Errors.Count > 0 ? Response.ErrorResponse(Errors) : new Response();
            }
            catch (Exception)
            {
                return Response.ErrorResponse(new List<string> {
                    "Unable to validate the request"
                });
            }
        }
    }
```

### AppConfig
Now we need to drill down to services or bussiness layer and pass required configurations. For that let's create a class representing our configuration. Use nested classes for organising it more maturely.

```csharp
    public class AppConfig
    {
        public Database Database { get; set; }
        public Security Security { get; set; }
    }

    public class Security
    {
        public string JwtSecret { get; set; }
    }

    public class Database
    {
        public string Server { get; set; }
        public string DatabaseName { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
        public bool IsTrustedConnection { get; set; }
    }
```
```json
  "Database": {
    "Server": "XXXXXXX\XXXXXX",
    "DatabaseName": "XXX",
    "IsTrustedConnection": true,
    "Username": "XXXX",
    "Password": "XXXX"
  },
  "Security": {
    "JWTPrivateKey": "Xuw2VWt59CExfba0o54NsKwt93AfCGlP"
  },
```
Use a static config builder to cast json as poco object (AppConfig). This is the AppConfigBuilder

```csharp
    public static class AppConfigBuilder
    {
        public static AppConfig Build()
        {
            var confg = new AppConfig
            {
                Database = new Database
                {
                    Server = Startup.Configuration.GetSection("Database:Server").Value,
                    DatabaseName = Startup.Configuration.GetSection("Database:DatabaseName").Value,
                    IsTrustedConnection = bool.Parse(Startup.Configuration.GetSection("Database:IsTrustedConnection").Value),
                    Username = Startup.Configuration.GetSection("Database:Username").Value,
                    Password = Startup.Configuration.GetSection("Database:Password").Value,
                },
                Security = new Security
                {
                    JwtSecret = Startup.Configuration.GetSection("Security:JWTPrivateKey").Value,
                }
            };
            return confg;
        }
    }
```
### Dependency Injection
Inject the services and a specific instance of the AppConfig as a Singleton. This makes life much easier
```csharp
            //Services DI
            services.AddHttpContextAccessor();

            var appConfig = AppConfigBuilder.Build();
            services.AddSingleton<AppConfig>(appConfig);

            services.AddSingleton<IAuthorityService, AuthorityService>();
            services.AddSingleton<IUserService, UserService>();
            services.AddSingleton<IProjectService, ProjectService>();
```

### JWT Authentication Middleware
Create a JWT MiddleWare
```csharp
    public class JwtMiddleware
    {
        private readonly RequestDelegate _next;

        public JwtMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task Invoke(HttpContext context, IUserService userService)
        {
            var token = context.Request.Headers["Authorization"].FirstOrDefault()?.Split(" ").Last();
            if (token != null) AttachUserToContext(context, userService, token);
            await _next(context);
        }

        private void AttachUserToContext(HttpContext context, IUserService userService, string token)
        {
            try
            {
                var tokenHandler = new JwtSecurityTokenHandler();
                var key = Encoding.ASCII.GetBytes(AppConfigBuilder.Build().Security.JwtSecret);
                tokenHandler.ValidateToken(token, new TokenValidationParameters
                {
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(key),
                    ValidateIssuer = false,
                    ValidateAudience = false,
                    ClockSkew = TimeSpan.Zero
                }, out SecurityToken validatedToken);
                var jwtToken = (JwtSecurityToken)validatedToken;
                var userId = int.Parse(jwtToken.Claims.First(x => x.Type == "id").Value);
                context.Items["User"] = userService.GetUserById(userId);
            }
            catch
            {
            }
        }
    }
```
And register it in Startup.cs
```csharp
app.UseMiddleware<JwtMiddleware>();
```

### JWT Authentication Filter
Implement an authentication filter for protected endpoints
```csharp
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
    public class AuthorizeAttribute : Attribute, IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            var user = (User)context.HttpContext.Items["User"];
            if (user == null)
            {
                context.Result = new JsonResult(new Response {
                    IsSuccess = false,
                    ResponseStatus = ResponseStatus.ERROR,
                    Message="Request terminated. Unauthorized access to protected resource.", 
                    Info=new List<string>() { 
                        "Verify the auth token sending through this request",
                        "Verify if your token is invalid or expired", 
                        "Request for a new token by logging in again" } 
                    })
                { StatusCode = StatusCodes.Status401Unauthorized };
            }
        }
    }
```


### JWT Token Generator
For login endpoint, Use the JWT token generator
```csharp
        private string GenerateJwtToken(User user)
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var key = Encoding.ASCII.GetBytes(appConfig.Security.JwtSecret);
            var tokenDescriptor = new SecurityTokenDescriptor
            {
                Subject = new ClaimsIdentity(new[] { new Claim("id", user.Id.ToString()) }),
                Expires = DateTime.UtcNow.AddDays(7),
                SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
            };
            var token = tokenHandler.CreateToken(tokenDescriptor);
            return tokenHandler.WriteToken(token);
        }
```

### Configure Entity Relationships
For multi one to one dependency cases, Configure the relationships using ForginKeys.
For Entities, Specify the props like this
```csharp
        public DateTime? UserCreatedOn { get; set; }
        public DateTime? UserUpdatedOn { get; set; }

        [ForeignKey("UserCreatedBy")]
        public User CreatedBy { get; set; }

        [ForeignKey("UserUpdatedBy")]
        public User UpdatedBy { get; set; }
```
And bind relationships like this
```csharp
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            //User
            modelBuilder.Entity<User>().HasOne(a => a.CreatedBy).WithOne().OnDelete(DeleteBehavior.Restrict);
            modelBuilder.Entity<User>().HasOne(a => a.UpdatedBy).WithOne().OnDelete(DeleteBehavior.Restrict);
            //Project
            modelBuilder.Entity<Project>().HasOne(a => a.CreatedBy).WithOne().OnDelete(DeleteBehavior.Restrict);
            modelBuilder.Entity<Project>().HasOne(a => a.UpdatedBy).WithOne().OnDelete(DeleteBehavior.Restrict);
        }
```
