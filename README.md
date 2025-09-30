using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Threading.Tasks;

namespace UserManagementApi
{
    // User Model with Validation
    public class User
    {
        public int Id { get; set; }

        [Required]
        [StringLength(50, MinimumLength = 2)]
        public string Name { get; set; }

        [Required]
        [EmailAddress]
        public string Email { get; set; }
    }

    // In-memory Repository
    public static class UserRepository
    {
        private static readonly List<User> Users = new()
        {
            new User { Id = 1, Name = "Alice", Email = "alice@example.com" },
            new User { Id = 2, Name = "Bob", Email = "bob@example.com" }
        };

        public static IEnumerable<User> GetAll() => Users;

        public static User GetById(int id) => Users.FirstOrDefault(u => u.Id == id);

        public static void Add(User user)
        {
            user.Id = Users.Any() ? Users.Max(u => u.Id) + 1 : 1;
            Users.Add(user);
        }

        public static void Update(int id, User updatedUser)
        {
            var existing = Users.FirstOrDefault(u => u.Id == id);
            if (existing != null)
            {
                existing.Name = updatedUser.Name;
                existing.Email = updatedUser.Email;
            }
        }

        public static void Delete(int id)
        {
            var user = Users.FirstOrDefault(u => u.Id == id);
            if (user != null)
                Users.Remove(user);
        }
    }

    // Middleware for Logging
    public class RequestLoggingMiddleware
    {
        private readonly RequestDelegate _next;

        public RequestLoggingMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            Console.WriteLine($"➡️ Request: {context.Request.Method} {context.Request.Path}");
            await _next(context);
            Console.WriteLine($"⬅️ Response: {context.Response.StatusCode}");
        }
    }

    // Middleware for Global Exception Handling
    public class ExceptionHandlingMiddleware
    {
        private readonly RequestDelegate _next;

        public ExceptionHandlingMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"❌ Exception: {ex.Message}");
                context.Response.StatusCode = 500;
                await context.Response.WriteAsJsonAsync(new { error = "An unexpected error occurred." });
            }
        }
    }

    // Minimal API Program.cs
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);
            builder.Services.AddEndpointsApiExplorer();
            builder.Services.AddSwaggerGen();

            var app = builder.Build();

            // Use Middleware
            app.UseMiddleware<ExceptionHandlingMiddleware>();
            app.UseMiddleware<RequestLoggingMiddleware>();

            // Swagger for API documentation
            app.UseSwagger();
            app.UseSwaggerUI();

            // CRUD Endpoints
            app.MapGet("/api/users", () => UserRepository.GetAll());

            app.MapGet("/api/users/{id}", (int id) =>
            {
                var user = UserRepository.GetById(id);
                return user is not null ? Results.Ok(user) : Results.NotFound();
            });

            app.MapPost("/api/users", (User user) =>
            {
                if (!MiniValidator.TryValidate(user, out var errors))
                    return Results.BadRequest(errors);

                UserRepository.Add(user);
                return Results.Created($"/api/users/{user.Id}", user);
            });

            app.MapPut("/api/users/{id}", (int id, User user) =>
            {
                if (!MiniValidator.TryValidate(user, out var errors))
                    return Results.BadRequest(errors);

                var existing = UserRepository.GetById(id);
                if (existing is null) return Results.NotFound();

                UserRepository.Update(id, user);
                return Results.NoContent();
            });

            app.MapDelete("/api/users/{id}", (int id) =>
            {
                var existing = UserRepository.GetById(id);
                if (existing is null) return Results.NotFound();

                UserRepository.Delete(id);
                return Results.NoContent();
            });

            app.Run();
        }
    }

    // Simple validation helper
    public static class MiniValidator
    {
        public static bool TryValidate(object obj, out Dictionary<string, string[]> errors)
        {
            var validationResults = new List<ValidationResult>();
            var context = new ValidationContext(obj);
            bool isValid = Validator.TryValidateObject(obj, context, validationResults, true);

            errors = validationResults
                .GroupBy(r => r.MemberNames.FirstOrDefault() ?? "")
                .ToDictionary(g => g.Key, g => g.Select(r => r.ErrorMessage).ToArray());

            return isValid;
        }
    }
}
