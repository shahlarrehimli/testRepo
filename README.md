# testRepo
tester

Winform/
using System.Windows.Forms;
using System;
using System.Net.Sockets;
using System.Threading;
using System.Collections.Generic;
using System.Text;
using System.Text.Json;

namespace WindowsFormsApp1
{
    partial class Form1
    {
        private System.ComponentModel.IContainer components = null;
        private System.Windows.Forms.DataGridView dataGridView1;
        private System.Windows.Forms.Button btnConnect;
        private System.Windows.Forms.Button btnStop;

        TcpClient client;
        NetworkStream stream;
        Thread receiveThread;
        bool receiving = false;
        protected override void Dispose(bool disposing)
        {
            if (disposing && (components != null))
            {
                components.Dispose();
            }
            base.Dispose(disposing);
        }
        private void InitializeComponent()
        {
            this.dataGridView1 = new System.Windows.Forms.DataGridView();
            this.btnConnect = new System.Windows.Forms.Button();
            this.btnStop = new System.Windows.Forms.Button();
            ((System.ComponentModel.ISupportInitialize)(this.dataGridView1)).BeginInit();
            this.SuspendLayout();

            this.dataGridView1.Anchor = ((System.Windows.Forms.AnchorStyles)(((System.Windows.Forms.AnchorStyles.Top | System.Windows.Forms.AnchorStyles.Bottom)
            | System.Windows.Forms.AnchorStyles.Left)));
            this.dataGridView1.Location = new System.Drawing.Point(12, 12);
            this.dataGridView1.Name = "dataGridView1";
            this.dataGridView1.Size = new System.Drawing.Size(500, 426);
            this.dataGridView1.TabIndex = 0;
            this.dataGridView1.AutoSizeColumnsMode = System.Windows.Forms.DataGridViewAutoSizeColumnsMode.Fill;

            this.btnConnect.Location = new System.Drawing.Point(530, 50);
            this.btnConnect.Name = "btnConnect";
            this.btnConnect.Size = new System.Drawing.Size(100, 40);
            this.btnConnect.TabIndex = 1;
            this.btnConnect.Text = "Connect";
            this.btnConnect.UseVisualStyleBackColor = true;
            this.btnConnect.Click += new System.EventHandler(this.btnConnect_Click);

            this.btnStop.Location = new System.Drawing.Point(530, 110);
            this.btnStop.Name = "btnStop";
            this.btnStop.Size = new System.Drawing.Size(100, 40);
            this.btnStop.TabIndex = 2;
            this.btnStop.Text = "Stop";
            this.btnStop.UseVisualStyleBackColor = true;
            this.btnStop.Click += new System.EventHandler(this.btnStop_Click);

            this.ClientSize = new System.Drawing.Size(650, 450);
            this.Controls.Add(this.dataGridView1);
            this.Controls.Add(this.btnConnect);
            this.Controls.Add(this.btnStop);
            this.Name = "Form1";
            this.Text = "WPF Data Viewer";
            ((System.ComponentModel.ISupportInitialize)(this.dataGridView1)).EndInit();
            this.ResumeLayout(false);
        }
        private void btnConnect_Click(object sender, EventArgs e)
        {
            client = new TcpClient();
            client.Connect("127.0.0.1", 5000); // WPF server
            stream = client.GetStream();
            receiving = true;

            receiveThread = new Thread(ReceiveData);
            receiveThread.IsBackground = true;
            receiveThread.Start();
        }
        private void btnStop_Click(object sender, EventArgs e)
        {
            receiving = false;
            if (client != null)
            {
                client.Close();
            }
        }
        private void ReceiveData()
        {
            byte[] buffer = new byte[4096];

            while (receiving)
            {
                try
                {
                    int bytesRead = stream.Read(buffer, 0, buffer.Length);
                    if (bytesRead == 0) continue;

                    string json = Encoding.UTF8.GetString(buffer, 0, bytesRead);

                    // Deserialize
                    var list = JsonSerializer.Deserialize<List<RowData>>(json);

                    // DataGridView update must be on UI thread
                    dataGridView1.Invoke((MethodInvoker)delegate
                    {
                        dataGridView1.DataSource = list;
                    });
                }
                catch { }
            }
        }
    }
}



//WPF///

using System;
using System.Net.Sockets;
using System.Net;
using System.Windows;
using System.Windows.Threading;
using WpfApp1.Models;
using System.Text;
using Newtonsoft.Json;

namespace WpfApp1
{
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        Random random = new Random();
        DispatcherTimer timer = new DispatcherTimer();
        const string chars = "AshnjwbdubySJIWNSQJNKDXSNCPQOWISMXjewioncdjkdsnakccdhbecd";

        TcpListener listener;
        bool clientConnected = false;
        TcpClient client;

        private System.Collections.Generic.List<RowData> currentData = new System.Collections.Generic.List<RowData>();
        private bool sendEnabled = false; // TCP göndərməni idarə edir
        public MainWindow()
        {
            InitializeComponent();

            timer.Interval=TimeSpan.FromSeconds(5);
            timer.Tick += Timer_Tick;

            GenerateData();
            timer.Start();
            StartServer();
        }
        private void StartServer()
        {
            listener = new TcpListener(IPAddress.Loopback, 5000); // localhost:5000
            listener.Start();
            listener.BeginAcceptTcpClient(AcceptClient, null);
        }

        private void AcceptClient(IAsyncResult ar)
        {
            client = listener.EndAcceptTcpClient(ar);
            clientConnected = true;
            listener.BeginAcceptTcpClient(AcceptClient, null);
            if(sendEnabled)
                SendDataToClient(currentData);
        }
        private void Send_Button(object sender, RoutedEventArgs e)
        {
            sendEnabled = true;
            SendDataToClient(currentData);
        }

        private void Stop_Button(object sender, RoutedEventArgs e)
        {
            sendEnabled = false;
        }
        private void Timer_Tick(object sender, EventArgs e)
        {
            GenerateData();
        }
        private void GenerateData()
        {
          currentData=new System.Collections.Generic.List<RowData>();
            
            for (int i = 1; i < 6; i++)
            {
                currentData.Add(new RowData
                {
                    Number = i,
                    Height = random.Next(1,99)/100.0,
                    Text = RandomTextGenerator(8)
                });
            }
            MyGrid.ItemsSource = currentData;

            if (sendEnabled)
                SendDataToClient(currentData);
        }
        private void SendDataToClient(System.Collections.Generic.List<RowData> list)
        {
            if (clientConnected && client != null)
            {
                try
                {
                    NetworkStream stream = client.GetStream();
                    string json = JsonConvert.SerializeObject(list);
                    byte[] data = Encoding.UTF8.GetBytes(json + "\n"); // \n ilə ayırırıq
                    stream.Write(data, 0, data.Length);
                }
                catch
                {
                    clientConnected = false;
                    client.Close();
                    listener.BeginAcceptTcpClient(AcceptClient, null);
                }
            }
        }
        private string RandomTextGenerator(int length)
        {
            char[] text=new char[length];
            for (int i = 0; i < length; i++)
            {
                text[i]= chars[random.Next(chars.Length)];
            }
            return new string(text);
        }
    }
}


model//

public class RowData
{
    public int Number { get; set; }
    public double Height { get; set; }
    public string Text { get; set; }
}
