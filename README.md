# ChatgptCloneAndoid
estudos sobre um clone do chatgpt para android
aplicativo Android que:

Faz chamadas para uma API de chat.
Salva o histórico de mensagens em um banco de dados local usando Room.
Exibe mensagens e respostas em uma RecyclerView.
Tem um tratamento de erros mais robusto.


Minimum API level: Selecione uma versão (ex.: API 21).


1. Adicionar Dependências
Abra o arquivo build.gradle (Module: app) e adicione as seguintes dependências:

groovy
implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
implementation 'androidx.recyclerview:recyclerview:1.2.1'
implementation 'androidx.room:room-runtime:2.4.2'
annotationProcessor 'androidx.room:room-compiler:2.4.2' 

2. Criar a Estrutura do Banco de Dados com Room
Crie um pacote chamado database.
Dentro de database, crie a classe MessageEntity.java:

import androidx.room.Entity;
import androidx.room.PrimaryKey;

@Entity(tableName = "messages")
public class MessageEntity {
    @PrimaryKey(autoGenerate = true)
    private int id;
    private String message;
    private String response;

    public MessageEntity(String message, String response) {
        this.message = message;
        this.response = response;
    }

    // Getters e Setters
    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getMessage() {
        return message;
    }

    public String getResponse() {
        return response;
    }
}

3.Crie uma interface MessageDao.java:

import androidx.room.Dao;
import androidx.room.Insert;
import androidx.room.Query;

import java.util.List;

@Dao
public interface MessageDao {
    @Insert
    void insert(MessageEntity message);

    @Query("SELECT * FROM messages ORDER BY id ASC")
    List<MessageEntity> getAllMessages();
}

4.Crie a classe de banco de dados AppDatabase.java:

import androidx.room.Database;
import androidx.room.Room;
import androidx.room.RoomDatabase;
import android.content.Context;

@Database(entities = {MessageEntity.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    private static AppDatabase instance;

    public abstract MessageDao messageDao();

    public static synchronized AppDatabase getInstance(Context context) {
        if (instance == null) {
            instance = Room.databaseBuilder(context.getApplicationContext(),
                    AppDatabase.class, "app_database")
                    .fallbackToDestructiveMigration()
                    .build();
        }
        return instance;
    }
}

5. Criar o Adapter para RecyclerView
Crie um pacote chamado adapter.
Dentro de adapter, crie a classe MessageAdapter.java:

import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;
import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;
import java.util.List;

public class MessageAdapter extends RecyclerView.Adapter<MessageAdapter.MessageViewHolder> {
    private List<MessageEntity> messages;

    public static class MessageViewHolder extends RecyclerView.ViewHolder {
        public TextView textMessage;
        public TextView textResponse;

        public MessageViewHolder(View view) {
            super(view);
            textMessage = view.findViewById(R.id.textMessage);
            textResponse = view.findViewById(R.id.textResponse);
        }
    }

    public MessageAdapter(List<MessageEntity> messages) {
        this.messages = messages;
    }

    @NonNull
    @Override
    public MessageViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.message_item, parent, false);
        return new MessageViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull MessageViewHolder holder, int position) {
        MessageEntity message = messages.get(position);
        holder.textMessage.setText(message.getMessage());
        holder.textResponse.setText(message.getResponse());
    }

    @Override
    public int getItemCount() {
        return messages.size();
    }

    public void setMessages(List<MessageEntity> messages) {
        this.messages = messages;
        notifyDataSetChanged();
    }
}

6.Crie um layout para o item da mensagem:
Crie um novo arquivo de layout chamado message_item.xml na pasta res/layout.
message_item.xml

<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="8dp">

    <TextView
        android:id="@+id/textMessage"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        android:textColor="#000"/>

    <TextView
        android:id="@+id/textResponse"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="14sp"
        android:textColor="#888"
        android:paddingTop="4dp"/>
</LinearLayout
  
7. Atualizar a Interface do Usuário para RecyclerView
Modifique activity_main.xml para incluir RecyclerView:
xml

<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText
        android:id="@+id/editTextMessage"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Digite sua mensagem"
        android:inputType="textMultiLine"
        android:gravity="top"
        android:minHeight="100dp"/>

    <Button
        android:id="@+id/buttonSend"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Enviar"
        android:layout_gravity="end"
        android:layout_marginTop="8dp"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewMessages"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:layout_marginTop="16dp"/>
</LinearLayout>

8.Atualize MainActivity.java para usar RecyclerView:

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;
import retrofit2.Call;
import retrofit2.Callback;
import retrofit2.Response;
import retrofit2.Retrofit;
import retrofit2.converter.gson.GsonConverterFactory;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {
    private EditText editTextMessage;
    private RecyclerView recyclerViewMessages;
    private MessageAdapter messageAdapter;
    private List<MessageEntity> messageList;
    private AppDatabase db;
    private ApiService apiService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        editTextMessage = findViewById(R.id.editTextMessage);
        recyclerViewMessages = findViewById(R.id.recyclerViewMessages);
        Button buttonSend = findViewById(R.id.buttonSend);

        // Inicializar o banco de dados
        db = AppDatabase.getInstance(this);
        messageList = new ArrayList<>();
        messageAdapter = new MessageAdapter(messageList);

        recyclerViewMessages.setLayoutManager(new LinearLayoutManager(this));
        recyclerViewMessages.setAdapter(messageAdapter);

        // Configurar Retrofit
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("https://api.openai.com/")
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        apiService = retrofit.create(ApiService.class);

        buttonSend.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String userMessage = editTextMessage.getText().toString();
                sendMessage(userMessage);
            }
        });

        loadMessages();
    }

    private void sendMessage(String message) {
        ChatRequest request = new ChatRequest(message);
        apiService.getChatResponse(request).enqueue(new Callback<ChatResponse>() {
            @Override
            public void onResponse(Call<ChatResponse> call, Response<ChatResponse> response) {
                if (response.isSuccessful() && response.body() != null) {
                    String chatResponse = response.body().getResponse();

                    // Salvar mensagem no banco de dados
                    MessageEntity messageEntity = new MessageEntity(message, chatResponse);
                    new Thread(() -> db.messageDao().insert(messageEntity)).start();

                    // Atualizar RecyclerView
                    loadMessages();
                } else {
                    showError("Erro: " + response.message());
                }
            }

            @Override
            public void onFailure(Call<ChatResponse> call, Throwable t) {
                showError(t instanceof IOException ? "Erro de rede. Verifique sua conexão." : "Erro inesperado: " + t.getMessage());
            }
        });
    }

    private void loadMessages() {
        new Thread(() -> {
            List<MessageEntity> messages = db.messageDao().getAllMessages();
            runOnUiThread(() -> messageAdapter.setMessages(messages));
        }).start();
    }

    private void showError(String error) {
        runOnUiThread(() -> {
            // Exibir erro (pode ser um Toast ou TextView)
            Toast.makeText(MainActivity.this, error, Toast.LENGTH_SHORT).show();
        });
    }
}

9. Configurar a Interface da API
Crie um pacote chamado api.
Dentro de api, crie a interface ApiService.java:

import retrofit2.Call;
import retrofit2.http.Body;
import retrofit2.http.POST;

public interface ApiService {
    @POST("v1/chat/completions") // Altere para o endpoint correto da sua API
    Call<ChatResponse> getChatResponse(@Body ChatRequest request);
}
10.Crie as classes ChatRequest.java e ChatResponse.java:


public class ChatRequest {
    private String message;

    public ChatRequest(String message) {
        this.message = message;
    }

    // Getter
    public String getMessage() {
        return message;
    }
}
ChatResponse.java

public class ChatResponse {
    private String response;

    public String getResponse() {
        return response;
    }
}
11. Testar o Aplicativo (cruze os dedos)
Execute o aplicativo em um (dispositivo ou) emulador Android.

Digite mensagens e verifique se o histórico é salvo e exibido corretamente.
Faz chamadas para uma API de chat.
Salva o histórico de mensagens em um banco de dados local usando Room.
Exibe mensagens e respostas em uma RecyclerView.
