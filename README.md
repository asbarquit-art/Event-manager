import java.io.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.*;
import java.util.stream.Collectors;

// --- MODELO ---

enum Categoria {
    FESTA, ESPORTE, SHOW, CONFERENCIA, OUTROS
}

class Usuario {
    private String nome;
    private String email;
    private String cidade;
    private List<Evento> eventosConfirmados = new ArrayList<>();

    public Usuario(String nome, String email, String cidade) {
        this.nome = nome;
        this.email = email;
        this.cidade = cidade;
    }

    public String getNome() { return nome; }
    public List<Evento> getEventosConfirmados() { return eventosConfirmados; }
}

class Evento implements Serializable {
    private String nome;
    private String endereco;
    private Categoria categoria;
    private LocalDateTime horario;
    private String descricao;

    public Evento(String nome, String endereco, Categoria categoria, LocalDateTime horario, String descricao) {
        this.nome = nome;
        this.endereco = endereco;
        this.categoria = categoria;
        this.horario = horario;
        this.descricao = descricao;
    }

    public String getNome() { return nome; }
    public LocalDateTime getHorario() { return horario; }

    public String Status() {
        LocalDateTime agora = LocalDateTime.now();
        if (horario.isBefore(agora.minusHours(3))) return "ENCERRADO"; // Exemplo: duração de 3h
        if (horario.isBefore(agora) && horario.plusHours(3).isAfter(agora)) return "OCORRENDO AGORA";
        return "FUTURO";
    }

    @Override
    public String toString() {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
        return String.format("[%s] %s | Categoria: %s | Local: %s | Horário: %s\nDesc: %s",
                Status(), nome.toUpperCase(), categoria, endereco, horario.format(dtf), descricao);
    }
}

// --- PERSISTÊNCIA E LOGICA (CONTROLLER) ---

class EventoManager {
    private List<Evento> eventos = new ArrayList<>();
    private final String ARQUIVO = "events.data";

    @SuppressWarnings("unchecked")
    public void carregarDados() {
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(ARQUIVO))) {
            eventos = (List<Evento>) ois.readObject();
        } catch (FileNotFoundException e) {
            System.out.println("Arquivo de dados não encontrado. Iniciando novo banco de dados.");
        } catch (Exception e) {
            System.out.println("Erro ao carregar dados: " + e.getMessage());
        }
    }

    public void salvarDados() {
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(ARQUIVO))) {
            oos.writeObject(eventos);
        } catch (IOException e) {
            System.out.println("Erro ao salvar dados: " + e.getMessage());
        }
    }

    public void adicionarEvento(Evento e) {
        eventos.add(e);
        salvarDados();
    }

    public List<Evento> getEventosOrdenados() {
        return eventos.stream()
                .sorted(Comparator.comparing(Evento::getHorario))
                .collect(Collectors.toList());
    }
}

// --- INTERFACE (VIEW) ---

public class SistemaEventos {
    private static Scanner sc = new Scanner(System.in);
    private static EventoManager manager = new EventoManager();
    private static Usuario usuarioLogado;
    private static DateTimeFormatter dtf = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");

    public static void main(String[] args) {
        manager.carregarDados();
        cadastrarUsuario();

        int opcao = -1;
        while (opcao != 0) {
            System.out.println("\n--- SISTEMA DE EVENTOS - " + usuarioLogado.getNome().toUpperCase() + " ---");
            System.out.println("1. Cadastrar Novo Evento");
            System.out.println("2. Consultar Eventos da Cidade");
            System.out.println("3. Meus Eventos Confirmados");
            System.out.println("4. Cancelar Participação");
            System.out.println("0. Sair");
            System.out.print("Escolha: ");
            
            try {
                opcao = Integer.parseInt(sc.nextLine());
                switch (opcao) {
                    case 1 -> menuCriarEvento();
                    case 2 -> menuListarEventos();
                    case 3 -> listarMeusEventos();
                    case 4 -> cancelarParticipacao();
                    case 0 -> System.out.println("Saindo...");
                    default -> System.out.println("Opção inválida.");
                }
            } catch (Exception e) {
                System.out.println("Erro na operação: " + e.getMessage());
            }
        }
    }

    private static void cadastrarUsuario() {
        System.out.println("--- CADASTRO DE USUÁRIO ---");
        System.out.print("Nome: "); String nome = sc.nextLine();
        System.out.print("Email: "); String email = sc.nextLine();
        System.out.print("Cidade: "); String cidade = sc.nextLine();
        usuarioLogado = new Usuario(nome, email, cidade);
    }

    private static void menuCriarEvento() {
        System.out.print("Nome do Evento: "); String nome = sc.nextLine();
        System.out.print("Endereço: "); String end = sc.nextLine();
        System.out.print("Descrição: "); String desc = sc.nextLine();
        System.out.println("Categorias: 1-FESTA, 2-ESPORTE, 3-SHOW, 4-CONFERENCIA, 5-OUTROS");
        int catIdx = Integer.parseInt(sc.nextLine()) - 1;
        Categoria cat = Categoria.values()[catIdx];
        
        System.out.print("Data e Hora (dd/MM/yyyy HH:mm): ");
        LocalDateTime data = LocalDateTime.parse(sc.nextLine(), dtf);

        manager.adicionarEvento(new Evento(nome, end, cat, data, desc));
        System.out.println("Evento cadastrado com sucesso!");
    }

    private static void menuListarEventos() {
        List<Evento> lista = manager.getEventosOrdenados();
        if (lista.isEmpty()) {
            System.out.println("Nenhum evento cadastrado.");
            return;
        }

        System.out.println("\n--- EVENTOS DISPONÍVEIS (Ordenados por Data) ---");
        for (int i = 0; i < lista.size(); i++) {
            System.out.println((i + 1) + ". " + lista.get(i));
            System.out.println("-------------------------------------------------");
        }

        System.out.print("Digite o número do evento para confirmar presença (ou 0 para voltar): ");
        int escolha = Integer.parseInt(sc.nextLine());
        if (escolha > 0 && escolha <= lista.size()) {
            Evento selecionado = lista.get(escolha - 1);
            if (!usuarioLogado.getEventosConfirmados().contains(selecionado)) {
                usuarioLogado.getEventosConfirmados().add(selecionado);
                System.out.println("Presença confirmada em: " + selecionado.getNome());
            } else {
                System.out.println("Você já confirmou presença neste evento.");
            }
        }
    }

    private static void listarMeusEventos() {
        System.out.println("\n--- MINHA AGENDA ---");
        if (usuarioLogado.getEventosConfirmados().isEmpty()) {
            System.out.println("Você não confirmou presença em nenhum evento.");
        } else {
            usuarioLogado.getEventosConfirmados().forEach(System.out::println);
        }
    }

    private static void cancelarParticipacao() {
        List<Evento> meus = usuarioLogado.getEventosConfirmados();
        if (meus.isEmpty()) {
            System.out.println("Agenda vazia.");
            return;
        }

        for (int i = 0; i < meus.size(); i++) {
            System.out.println((i + 1) + ". " + meus.get(i).getNome());
        }
        System.out.print("Selecione o número para cancelar: ");
        int idx = Integer.parseInt(sc.nextLine()) - 1;
        if (idx >= 0 && idx < meus.size()) {
            System.out.println("Participação cancelada em: " + meus.remove(idx).getNome());
        }
    }
}
