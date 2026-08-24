# LAB1 MICROS - Código IDE

#include <Keypad.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

byte filas[4] = {
  13, 14, 27, 26
};

byte columnas[4] = {
  25, 33, 32, 23
};

char teclas[4][4] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

Keypad teclado = Keypad(
  makeKeymap(teclas),
  filas,
  columnas,
  4,
  4
);

const int PIN_SDA = 21;
const int PIN_SCL = 22;

LiquidCrystal_I2C lcd(0x27, 16, 2);

const int PIN_IN1 = 18;
const int PIN_IN2 = 19;
const int PIN_IN3 = 17;
const int PIN_IN4 = 16;

const int PIN_HOME = 4;

const byte SECUENCIA[8][4] = {

  {1, 0, 0, 0},
  {1, 1, 0, 0},
  {0, 1, 0, 0},
  {0, 1, 1, 0},
  {0, 0, 1, 0},
  {0, 0, 1, 1},
  {0, 0, 0, 1},
  {1, 0, 0, 1}

};

int indiceSecuencia = 0;

long posicion = 0;

// escala de velocidad 1 a 10
int velocidad = 5;

String texto = "";

enum Modo {

  MENU,
  INGRESAR_ANGULO,
  INGRESAR_VELOCIDAD

};

Modo modo = MENU;

void setup() {

  Serial.begin(115200);

  // Pines ULN2003

  pinMode(PIN_IN1, OUTPUT);
  pinMode(PIN_IN2, OUTPUT);
  pinMode(PIN_IN3, OUTPUT);
  pinMode(PIN_IN4, OUTPUT);

  apagarMotor();
  pinMode(PIN_HOME, INPUT);

  // LCD
  Wire.begin(PIN_SDA, PIN_SCL);

  lcd.init();
  lcd.backlight();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Inicializando");

  delay(500);

  if (!buscarHome()) {

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("ERROR SENSOR");
    lcd.setCursor(0, 1);
    lcd.print("Revisar HOME");

    Serial.println(
      "ERROR: No se encontro HOME."
    );

    apagarMotor();

    while (true) {
      delay(100);
    }
  }

  mostrarMenu();
}

void loop() {

  char tecla = teclado.getKey();

  if (!tecla) {
    return;
  }

  // MENU

  if (modo == MENU) {

    if (tecla == 'A') {

      texto = "";
      modo = INGRESAR_ANGULO;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Angulo 0-359:");

    }

    else if (tecla == 'B') {
      texto = "";
      modo = INGRESAR_VELOCIDAD;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Velocidad 1-10:");

    }
  }

  // INGRESAR ANGULO

  else if (modo == INGRESAR_ANGULO) {
    if (tecla >= '0' &&
        tecla <= '9') {
      if (texto.length() < 3) {
        texto += tecla;
        mostrarTextoEntrada();
      }
    }

    else if (tecla == '*') {
      texto = "";
      mostrarTextoEntrada();
    }

    else if (tecla == 'D') {
      modo = MENU;
      mostrarMenu();
    }

    else if (tecla == '#') {
      if (texto.length() == 0) {
        return;
      }

      int angulo = texto.toInt();
      if (angulo >= 0 &&
          angulo <= 359) {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Moviendo a:");
        lcd.setCursor(0, 1);
        lcd.print(angulo);
        lcd.print(" grados");
        moverAAngulo(angulo);

        delay(300);

        modo = MENU;
        mostrarMenu();
      }

      else {

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Angulo invalido");
        lcd.setCursor(0, 1);
        lcd.print("Use 0-359");
        delay(1000);
        texto = "";
        lcd.clear();
        lcd.print("Angulo 0-359:");
      }
    }
  }

  // INGRESAR VELOCIDAD

  else if (modo == INGRESAR_VELOCIDAD) {
    if (tecla >= '0' &&
        tecla <= '9') {
      if (texto.length() < 2) {
        texto += tecla;
        mostrarTextoEntrada();
      }
    }

    else if (tecla == '*') {
      texto = "";
      mostrarTextoEntrada();
    }

    else if (tecla == 'D') {
      modo = MENU;
      mostrarMenu();
    }

    else if (tecla == '#') {
      if (texto.length() == 0) {
        return;
      }

      int nuevaVelocidad =
        texto.toInt();

      if (nuevaVelocidad >= 1 &&
          nuevaVelocidad <= 10) {

        velocidad =
          nuevaVelocidad;

        modo = MENU;
        mostrarMenu();
      }

      else {

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Vel invalida");
        lcd.setCursor(0, 1);
        lcd.print("Use 1-10");
        delay(1000);

        texto = "";
        lcd.clear();

        lcd.print(
          "Velocidad 1-10:"
        );
      }
    }
  }
}
// SENSOR HOME

bool homeActivo() {

  return digitalRead(PIN_HOME)
         == HOME_ACTIVO;
}

// SECUENCIA AL ULN2003

void aplicarSecuencia(int indice) {

  digitalWrite(
    PIN_IN1,
    SECUENCIA[indice][0]
  );

  digitalWrite(
    PIN_IN2,
    SECUENCIA[indice][1]
  );

  digitalWrite(
    PIN_IN3,
    SECUENCIA[indice][2]
  );

  digitalWrite(
    PIN_IN4,
    SECUENCIA[indice][3]
  );
}

void apagarMotor() {

  digitalWrite(PIN_IN1, LOW);
  digitalWrite(PIN_IN2, LOW);
  digitalWrite(PIN_IN3, LOW);
  digitalWrite(PIN_IN4, LOW);
}

// GENERAR UN MEDIO PASO

void generarPaso(
  bool horario,
  bool contarPosicion,
  int esperaUs
) {

  // Invertir sentido
  bool sentidoFisico;

  if (INVERTIR_SENTIDO_MOTOR) {
    sentidoFisico = !horario;
  }
  else {
    sentidoFisico = horario;
  }

  if (sentidoFisico) {

    indiceSecuencia++;

    if (indiceSecuencia >= 8) {
      indiceSecuencia = 0;
    }
  }

  else {

    indiceSecuencia--;

    if (indiceSecuencia < 0) {
      indiceSecuencia = 7;
    }
  }

  // Activar bobinas

  aplicarSecuencia(
    indiceSecuencia
  );

  delayMicroseconds(
    esperaUs
  );

  if (contarPosicion) {
    if (horario) {
      posicion++;
    }
    else {
      posicion--;
    }
    posicion %=
      PASOS_POR_VUELTA;

    if (posicion < 0) {
      posicion +=
        PASOS_POR_VUELTA;
    }
  }
}

// PASO A VELOCIDAD SELECCIONADA

void paso(bool horario) {

  int espera = map(
    velocidad,
    1,
    10,
    6000,
    1200
  );

  generarPaso(
    horario,
    true,
    espera
  );
}

bool moverHastaEstadoHome(
  bool estadoDeseado,
  bool horario,
  long maxPasos,
  int esperaUs
) {

  for (
    long i = 0;
    i < maxPasos;
    i++
  ) {

    if (
      homeActivo()
      == estadoDeseado
    ) {

      return true;
    }

    generarPaso(
      horario,
      false,
      esperaUs
    );
  }

  return
    homeActivo()
    == estadoDeseado;
}

bool buscarHome() {

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("Buscando HOME");

  Serial.println(
    "Iniciando busqueda HOME..."
  );

  if (homeActivo()) {

    lcd.setCursor(0, 1);
    lcd.print("Liberando...");

    bool libero =
      moverHastaEstadoHome(

        false,
        !DIRECCION_HOME,
        PASOS_POR_VUELTA,
        3000

      );

    if (!libero) {

      apagarMotor();

      return false;
    }
  }

  lcd.setCursor(0, 1);
  lcd.print("Aproximando... ");

  bool encontro =
    moverHastaEstadoHome(

      true,
      DIRECCION_HOME,
      PASOS_POR_VUELTA * 2,
      2200

    );

  if (!encontro) {
    apagarMotor();
    return false;
  }

  bool libero =
    moverHastaEstadoHome(

      false,
      !DIRECCION_HOME,
      PASOS_POR_VUELTA / 2,
      3500
    );

  if (!libero) {
    apagarMotor();
    return false;
  }

  for (int i = 0; i < 15; i++) {
    generarPaso(
      !DIRECCION_HOME,
      false,
      4000

    );
  }
  lcd.setCursor(0, 1);
  lcd.print("Ajuste fino...  ");

  encontro =
    moverHastaEstadoHome(
      true,
      DIRECCION_HOME,
      PASOS_POR_VUELTA,
      5000
    );

  if (!encontro) {
    apagarMotor();
    return false;
  }
  posicion = 0;

  Serial.println(
    "HOME encontrado."
  );

  Serial.println(
    "Posicion = 0 grados."
  );


  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("HOME OK");
  lcd.setCursor(0, 1);
  lcd.print("Posicion: 0");

  delay(1000);
  return true;
}
void moverAAngulo(int grados) {

  long objetivo =
    lround(
      grados *
      (double)PASOS_POR_VUELTA /
      360.0
    );

  objetivo %=
    PASOS_POR_VUELTA;

  long diferencia =
    objetivo - posicion;

  // movimiento horario

  if (diferencia > 0) {

    for (
      long i = 0;
      i < diferencia;
      i++
    ) {

      paso(true);
    }
  }
  // movimiento antihorario

  else if (diferencia < 0) {

    for (
      long i = 0;
      i < -diferencia;
      i++
    ) {

      paso(false);
    }
  }
}

// LCD
void mostrarTextoEntrada() {

  lcd.setCursor(0, 1);
  lcd.print(
    "                "
  );

  lcd.setCursor(0, 1);
  lcd.print(texto);
}

// MENU

void mostrarMenu() {

  int grados =
    (int)(

      posicion *
      360.0 /
      PASOS_POR_VUELTA

    );

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Pos:");
  lcd.print(grados);
  lcd.print(" Vel:");
  lcd.print(velocidad);


  lcd.setCursor(0, 1);

  lcd.print(
    "A=pos B=vel"
  );
}
