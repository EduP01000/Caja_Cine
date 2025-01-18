using System;
using System.Collections.Generic;
using System.Data.SqlClient;

public class CinePOS
{
    private string connectionString = @"Data Source=DESKTOP-MVO24EG\SQLEXPRESS;AttachDbFilename=""C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\DATA\CineDB.mdf"";Integrated Security=True;Encrypt=True;Trust Server Certificate=True";

    private string proyeccionSeleccionada;
    private int asientoSeleccionado;
    private decimal precioAsiento;
    private List<(string Nombre, int Cantidad, decimal PrecioTotal)> snacksSeleccionados = new List<(string, int, decimal)>();
    private List<(string Nombre, int Cantidad, decimal PrecioTotal)> combosSeleccionados = new List<(string, int, decimal)>();

    // Método para iniciar sesión
    public bool LoginUsuario()
    {
        Console.WriteLine("Ingrese su ID de Usuario:");
        string idUsuario = Console.ReadLine();

        Console.WriteLine("Ingrese su Clave:");
        string clave = Console.ReadLine();

        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            try
            {
                connection.Open();
                string query = "SELECT COUNT(*) FROM Usuarios WHERE id_Usuario = @idUsuario AND Clave = @clave";
                using (SqlCommand command = new SqlCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@idUsuario", idUsuario);
                    command.Parameters.AddWithValue("@clave", clave);

                    int count = (int)command.ExecuteScalar();
                    if (count > 0)
                    {
                        Console.WriteLine("Login exitoso.");
                        return true;
                    }
                    else
                    {
                        Console.WriteLine("Credenciales incorrectas. Intente de nuevo.");
                        return false;
                    }
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Ocurrió un error: {ex.Message}");
                return false;
            }
        }
    }

    // Método para mostrar las proyecciones disponibles y seleccionar una
    public int SeleccionarProyeccion()
    {
        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            try
            {
                connection.Open();
                string query = "SELECT id_Proyeccion, Pelicula, id_Sala, FechaInicio, FechaFin, HoraInicio, HoraFin FROM Proyecciones";
                using (SqlCommand command = new SqlCommand(query, connection))
                using (SqlDataReader reader = command.ExecuteReader())
                {
                    Console.WriteLine("Proyecciones disponibles:");
                    while (reader.Read())
                    {
                        Console.WriteLine(
                            $"ID: {reader["id_Proyeccion"]}, " +
                            $"Película: {reader["Pelicula"]}, " +
                            $"Sala: {reader["id_Sala"]}, " +
                            $"Inicio: {reader["FechaInicio"]}, " +
                            $"Fin: {reader["FechaFin"]}, " +
                            $"Hora Inicio: {reader["HoraInicio"]}, " +
                            $"Hora Fin: {reader["HoraFin"]}");
                    }
                }
                Console.WriteLine("\nIngrese el ID de la proyección que desea seleccionar:");
                int idProyeccion = int.Parse(Console.ReadLine());

                query = "SELECT Pelicula FROM Proyecciones WHERE id_Proyeccion = @idProyeccion";
                using (SqlCommand command = new SqlCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@idProyeccion", idProyeccion);
                    proyeccionSeleccionada = command.ExecuteScalar().ToString();
                }

                return idProyeccion;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error al obtener las proyecciones: {ex.Message}");
                return -1;
            }
        }
    }

    // Método para seleccionar un asiento válido
    public void SeleccionarAsiento(int idProyeccion)
    {
        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            try
            {
                connection.Open();
                string querySala = "SELECT id_Sala FROM Proyecciones WHERE id_Proyeccion = @idProyeccion";
                int idSala;

                using (SqlCommand command = new SqlCommand(querySala, connection))
                {
                    command.Parameters.AddWithValue("@idProyeccion", idProyeccion);
                    idSala = (int)command.ExecuteScalar();
                }

                string queryAsientos = "SELECT No_Asiento, Estado, Precio FROM DetalleAsiento JOIN TipoAsiento ON DetalleAsiento.Tipo = TipoAsiento.Tipo WHERE id_Sala = @idSala AND Estado = 'Disponible'";
                using (SqlCommand command = new SqlCommand(queryAsientos, connection))
                {
                    command.Parameters.AddWithValue("@idSala", idSala);
                    using (SqlDataReader reader = command.ExecuteReader())
                    {
                        Console.WriteLine("Asientos disponibles:");
                        while (reader.Read())
                        {
                            Console.WriteLine($"Asiento: {reader["No_Asiento"]}, Precio: {reader["Precio"]}");
                        }
                    }
                }
                Console.WriteLine("\nIngrese el número del asiento que desea seleccionar:");
                asientoSeleccionado = int.Parse(Console.ReadLine());

                string queryPrecioAsiento = "SELECT Precio FROM DetalleAsiento JOIN TipoAsiento ON DetalleAsiento.Tipo = TipoAsiento.Tipo WHERE id_Sala = @idSala AND No_Asiento = @noAsiento AND Estado = 'Disponible'";
                using (SqlCommand command = new SqlCommand(queryPrecioAsiento, connection))
                {
                    command.Parameters.AddWithValue("@idSala", idSala);
                    command.Parameters.AddWithValue("@noAsiento", asientoSeleccionado);
                    precioAsiento = (decimal)command.ExecuteScalar();
                }

                Console.WriteLine($"Asiento seleccionado: {asientoSeleccionado}, Precio: {precioAsiento}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error al seleccionar el asiento: {ex.Message}");
            }
        }
    }

    // Método para realizar pedidos de snacks o combos
    public void RealizarPedido()
    {
        bool continuarOrdenando = true;

        while (continuarOrdenando)
        {
            Console.WriteLine("\n¿Desea ordenar Snacks o Combos? (Escriba 'Snacks' o 'Combos')");
            string opcion = Console.ReadLine();

            if (opcion.Equals("Snacks", StringComparison.OrdinalIgnoreCase))
            {
                PedirSnacks();
            }
            else if (opcion.Equals("Combos", StringComparison.OrdinalIgnoreCase))
            {
                PedirCombos();
            }
            else
            {
                Console.WriteLine("Opción no válida. Intente de nuevo.");
                continue;
            }

            Console.WriteLine("¿Desea seguir ordenando? (Escriba 'Sí' o 'No')");
            continuarOrdenando = Console.ReadLine().Equals("Sí", StringComparison.OrdinalIgnoreCase);
        }
    }

    // Método para pedir snacks
    private void PedirSnacks()
    {
        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            connection.Open();
            string query = "SELECT id_Snack, Snack, Precio FROM Snacks";

            using (SqlCommand command = new SqlCommand(query, connection))
            using (SqlDataReader reader = command.ExecuteReader())
            {
                Console.WriteLine("Snacks disponibles:");
                while (reader.Read())
                {
                    Console.WriteLine($"ID: {reader["id_Snack"]}, Snack: {reader["Snack"]}, Precio: {reader["Precio"]}");
                }
            }
        }
        Console.WriteLine("\nIngrese el ID del snack que desea:");
        int idSnack = int.Parse(Console.ReadLine());

        Console.WriteLine("Ingrese la cantidad:");
        int cantidad = int.Parse(Console.ReadLine());

        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            connection.Open();
            string query = "SELECT Snack, Precio FROM Snacks WHERE id_Snack = @idSnack";
            using (SqlCommand command = new SqlCommand(query, connection))
            {
                command.Parameters.AddWithValue("@idSnack", idSnack);
                using (SqlDataReader reader = command.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        string nombreSnack = reader["Snack"].ToString();
                        decimal precioSnack = (decimal)reader["Precio"];
                        decimal precioTotal = precioSnack * cantidad;

                        snacksSeleccionados.Add((nombreSnack, cantidad, precioTotal));
                        Console.WriteLine($"Has ordenado {cantidad} de {nombreSnack}, Total: {precioTotal}");
                    }
                }
            }
        }
    }

    // Método para pedir combos
    private void PedirCombos()
    {
        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            connection.Open();
            string query = "SELECT id_Combo, Nombre, Precio FROM Combos";

            using (SqlCommand command = new SqlCommand(query, connection))
            using (SqlDataReader reader = command.ExecuteReader())
            {
                Console.WriteLine("Combos disponibles:");
                while (reader.Read())
                {
                    Console.WriteLine($"ID: {reader["id_Combo"]}, Combo: {reader["Nombre"]}, Precio: {reader["Precio"]}");
                }
            }
        }
        Console.WriteLine("\nIngrese el ID del combo que desea:");
        int idCombo = int.Parse(Console.ReadLine());

        Console.WriteLine("Ingrese la cantidad:");
        int cantidad = int.Parse(Console.ReadLine());

        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            connection.Open();
            string query = "SELECT Nombre, Precio FROM Combos WHERE id_Combo = @idCombo";
            using (SqlCommand command = new SqlCommand(query, connection))
            {
                command.Parameters.AddWithValue("@idCombo", idCombo);
                using (SqlDataReader reader = command.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        string nombreCombo = reader["Nombre"].ToString();
                        decimal precioCombo = (decimal)reader["Precio"];
                        decimal precioTotal = precioCombo * cantidad;

                        combosSeleccionados.Add((nombreCombo, cantidad, precioTotal));
                        Console.WriteLine($"Has ordenado {cantidad} de {nombreCombo}, Total: {precioTotal}");
                    }
                }
            }
        }
    }

    // Método para mostrar el resumen final
    public void MostrarResumen()
    {
        Console.WriteLine("\n--- Resumen de la Orden ---");
        Console.WriteLine($"Proyección seleccionada: {proyeccionSeleccionada}");
        Console.WriteLine($"Asiento seleccionado: {asientoSeleccionado}, Precio: {precioAsiento}");

        Console.WriteLine("\nSnacks:");
        decimal totalSnacks = 0;
        foreach (var snack in snacksSeleccionados)
        {
            Console.WriteLine($"{snack.Nombre} x{snack.Cantidad}, Total: {snack.PrecioTotal}");
            totalSnacks += snack.PrecioTotal;
        }

        Console.WriteLine("\nCombos:");
        decimal totalCombos = 0;
        foreach (var combo in combosSeleccionados)
        {
            Console.WriteLine($"{combo.Nombre} x{combo.Cantidad}, Total: {combo.PrecioTotal}");
            totalCombos += combo.PrecioTotal;
        }

        decimal granTotal = totalSnacks + totalCombos + precioAsiento;
        Console.WriteLine($"\nTotal a pagar: {granTotal}");
    }
}

public class Program
{
    public static void Main(string[] args)
    {
        CinePOS cinePOS = new CinePOS();

        if (cinePOS.LoginUsuario())
        {
            int idProyeccion = cinePOS.SeleccionarProyeccion();
            cinePOS.SeleccionarAsiento(idProyeccion);
            cinePOS.RealizarPedido();
            cinePOS.MostrarResumen();
        }
    }
}
