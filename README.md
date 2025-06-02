# java-game
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.sql.*;
import java.util.*;
import javax.swing.Timer;

public class CountryGuessDBGameGUI extends JFrame {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/country_game";
    private static final String DB_USER = "root";
    private static final String DB_PASS = "India@123";

    private final Map<String, String> countryHints = new HashMap<>();
    private int score = 0;
    private int highestScore = 0;
    private int wrongAttempts = 0;
    private final int MAX_WRONG_ATTEMPTS = 1;
    private String selectedCountry;

    private JLabel hintLabel;
    private JRadioButton[] optionButtons = new JRadioButton[4];
    private ButtonGroup optionsGroup = new ButtonGroup();
    private JButton submitButton;
    private JLabel feedbackLabel;
    private JLabel scoreLabel;
    private JLabel timerLabel;
    private JButton nextButton;
    private JButton exitButton;
    private JButton scoreButton;
    private JButton highScoreButton;

    private Timer questionTimer;
    private int timeRemaining = 10;
    private final Random random = new Random();
    private String username;
    private int globalHighScore = 0;

    public CountryGuessDBGameGUI() {
        // Get username from the user
        username = JOptionPane.showInputDialog(this, "Enter your username:", "Welcome", JOptionPane.QUESTION_MESSAGE);
        if (username == null || username.trim().isEmpty()) {
            JOptionPane.showMessageDialog(this, "❗ Username is required. Exiting.");
            System.exit(0);
        }

        setTitle("Country Guessing Game - Player: " + username);
        setSize(1000, 1000);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        if (!loadHintsFromDatabase()) {
            JOptionPane.showMessageDialog(this, "❌ Failed to load country hints from DB.", "Error", JOptionPane.ERROR_MESSAGE);
            System.exit(1);
        }

        initUI();
        loadNextQuestion();
        getGlobalHighScore(); // Get the global high score from DB
    }

    private void initUI() {
        Font labelFont = new Font("SansSerif", Font.BOLD, 16);
        Font optionFont = new Font("SansSerif", Font.PLAIN, 16);

        hintLabel = new JLabel("Hint: ");
        hintLabel.setFont(labelFont);

        feedbackLabel = new JLabel(" ");
        feedbackLabel.setFont(labelFont);

        scoreLabel = new JLabel("Score: 0");
        scoreLabel.setFont(labelFont);

        timerLabel = new JLabel("Time left: 10s");
        timerLabel.setFont(labelFont);

        JPanel optionsPanel = new JPanel(new GridLayout(3, 1, 5, 5)); // Vertical spacing
        for (int i = 0; i < 4; i++) {
            optionButtons[i] = new JRadioButton();
            optionButtons[i].setFont(optionFont);
            optionsGroup.add(optionButtons[i]);
            optionsPanel.add(optionButtons[i]);
        }

        submitButton = new JButton("Submit");
        nextButton = new JButton("Next");
        exitButton = new JButton("Exit");
        scoreButton = new JButton("Score");
        highScoreButton = new JButton("Global High Score");

        submitButton.setFont(labelFont);
        nextButton.setFont(labelFont);
        exitButton.setFont(labelFont);
        scoreButton.setFont(labelFont);
        highScoreButton.setFont(labelFont);

        submitButton.addActionListener(e -> checkAnswer());
        nextButton.addActionListener(e -> loadNextQuestion());
        exitButton.addActionListener(e -> System.exit(0));
        scoreButton.addActionListener(e -> showHighestScore());
        highScoreButton.addActionListener(e -> showGlobalHighScore());

        JPanel mainPanel = new JPanel();
        mainPanel.setLayout(new GridLayout(12, 1, 5, 5));
        mainPanel.setBorder(BorderFactory.createEmptyBorder(0, 1, 0, 0));

        mainPanel.add(hintLabel);
        mainPanel.add(optionsPanel);
        mainPanel.add(submitButton);
        mainPanel.add(feedbackLabel);
        mainPanel.add(scoreLabel);
        mainPanel.add(timerLabel);
        mainPanel.add(nextButton);
        mainPanel.add(scoreButton);
        mainPanel.add(highScoreButton);
        mainPanel.add(exitButton);

        add(mainPanel);
    }

    private void getGlobalHighScore() {
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT MAX(score) AS high_score FROM high_scores")) {

            if (rs.next()) {
                globalHighScore = rs.getInt("high_score");
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void showGlobalHighScore() {
        JOptionPane.showMessageDialog(this,
                "🎯 Global Highest Score: " + globalHighScore,
                "Global High Score", JOptionPane.INFORMATION_MESSAGE);
    }

    private boolean loadHintsFromDatabase() {
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT country, hint FROM country_hints")) {

            while (rs.next()) {
                String country = rs.getString("country").trim();
                String hint = rs.getString("hint").trim();
                countryHints.put(country, hint);
            }
            return !countryHints.isEmpty();
        } catch (SQLException e) {
            e.printStackTrace();
            return false;
        }
    }

    private void updateGlobalHighScore(int score) {
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             PreparedStatement stmt = conn.prepareStatement("INSERT INTO high_scores (username, score) VALUES (?, ?)")) {

            stmt.setString(1, username);
            stmt.setInt(2, score);
            stmt.executeUpdate();

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void loadNextQuestion() {
        if (questionTimer != null && questionTimer.isRunning()) {
            questionTimer.stop();
        }

        optionsGroup.clearSelection();
        feedbackLabel.setText(" ");
        submitButton.setEnabled(true);

        ArrayList<String> countries = new ArrayList<>(countryHints.keySet());
        selectedCountry = countries.get(random.nextInt(countries.size()));
        String hint = countryHints.get(selectedCountry);
        hintLabel.setText("Hint: " + hint);

    
        Set<String> options = new LinkedHashSet<>();
        options.add(selectedCountry);
        while (options.size() < 4) {
            String randomCountry = countries.get(random.nextInt(countries.size()));
            options.add(randomCountry);
        }

        ArrayList<String> shuffledOptions = new ArrayList<>(options);
        Collections.shuffle(shuffledOptions);

        for (int i = 0; i < 4; i++) {
            optionButtons[i].setText(shuffledOptions.get(i));
            optionButtons[i].setEnabled(true);
        }

        startTimer();
    }

    private void startTimer() {
        timeRemaining = 10;
        timerLabel.setText("Time left: 10s");

        questionTimer = new Timer(1000, new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                timeRemaining--;
                timerLabel.setText("Time left: " + timeRemaining + "s");

                if (timeRemaining <= 0) {
                    questionTimer.stop();
                    JOptionPane.showMessageDialog(CountryGuessDBGameGUI.this,
                            "⏰ Time's up!\nFinal Score: " + score +
                            "\n🎯 Global Highest Score: " + globalHighScore,
                            "Time Up!", JOptionPane.INFORMATION_MESSAGE);
                    updateGlobalHighScore(score);  // Save the score
                    System.exit(0);
                }
            }
        });
        questionTimer.start();
    }

    private void checkAnswer() {
        if (questionTimer != null && questionTimer.isRunning()) {
            questionTimer.stop();
        }

        String guess = null;
        for (JRadioButton button : optionButtons) {
            if (button.isSelected()) {
                guess = button.getText();
                break;
            }
        }

        if (guess == null) {
            feedbackLabel.setText("⚠️ Please select an option.");
            return;
        }

        if (guess.equalsIgnoreCase(selectedCountry)) {
            feedbackLabel.setText("✅ Correct! The country is " + selectedCountry + ".");
            score++;
            highestScore = Math.max(highestScore, score);
            wrongAttempts = 0;
        } else {
            feedbackLabel.setText("❌ Incorrect. The correct answer was: " + selectedCountry + ".");
            wrongAttempts++;
            if (wrongAttempts >= MAX_WRONG_ATTEMPTS) {
                JOptionPane.showMessageDialog(this,
                        "❌ Game Over! Only one mistake allowed.\nFinal Score: " + score +
                        "\n🎯 Global Highest Score: " + globalHighScore,
                        "Game Over", JOptionPane.INFORMATION_MESSAGE);
                updateGlobalHighScore(score);  // Save the score
                // System.exit(0);
            }
        }

        
        if (score > globalHighScore) {
            globalHighScore = score;
            updateGlobalHighScore(score);
        }

        scoreLabel.setText("Score: " + score);
        submitButton.setEnabled(false); 
    }

    private void showHighestScore() {
        JOptionPane.showMessageDialog(this,
                "🎯 Your Highest Score: " + highestScore,
                "Your Highest Score", JOptionPane.INFORMATION_MESSAGE);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new CountryGuessDBGameGUI().setVisible(true));
    }
}
