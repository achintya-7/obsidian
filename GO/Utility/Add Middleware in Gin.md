A [[Utility]] in Gin is to use middleware
Firstly make an interface with the required functions. Its something like an abstract class.
```go
type Maker interface {
    CreateToken(email string, phone string, duration time.Duration) (string, error)
    VerifyToken(token string) (*Payload, error)
}
```

Make a simple PasetoMaker struct in another file and return a Maker interface along with error
```go
type PasetoMaker struct {
    paseto       *paseto.V2
    symmetricKey []byte
}

func NewPasetoMaker(symmetricKey string) (Maker, error) {
    if len(symmetricKey) != chacha20poly1305.KeySize {
        return nil, errors.New("symmetric key is empty")
    }

    maker := &PasetoMaker{
        paseto:       paseto.NewV2(),
        symmetricKey: []byte(symmetricKey),
    }

    return maker, nil
}

func (maker *PasetoMaker) CreateToken(email string, phone string, duration time.Duration) (string, error) {
    payLoad, err := NewPayload(email, phone, duration)
    if err != nil {
        return "", err
    }

    token, err := maker.paseto.Encrypt(maker.symmetricKey, payLoad, nil)
    if err != nil {
        return "", err
    }

    return token, nil
}

func (maker *PasetoMaker) VerifyToken(token string) (*Payload, error) {
    payload := &Payload{}

	err := maker.paseto.Decrypt(token, maker.symmetricKey, payload, nil)
    if err != nil {
        return nil, ErrInvalidToken
    }

    err = payload.Valid()
    if err != nil {
        return nil, err
    }

    return payload, nil
}
```

Now make a simple Payload Struct and create a New Payload function
```go
var (
    ErrInvalidToken = errors.New("invalid token")
    ErrExpiredToken = errors.New("expired token")
)

type Payload struct {
    ID        uuid.UUID `json:"id"`
    Phone     string    `json:"phone"`
    Email     string    `json:"email"`
    IssuedAt  time.Time `json:"issued_at"`
    ExpiredAt time.Time `json:"expired_at"`
}

func NewPayload(email string, phone string, duration time.Duration) (*Payload, error) {
    tokenID, err := uuid.NewRandom()
    if err != nil {
        return nil, err
    }

    payload := &Payload{
        ID:        tokenID,
        Email:     email,
        Phone:     phone,
        IssuedAt:  time.Now(),
        ExpiredAt: time.Now().Add(duration),
    }
    return payload, nil
}

func (payload *Payload) Valid() error {
    if time.Now().After(payload.ExpiredAt) {
        return ErrExpiredToken
    }
    return nil
}
```

Now make a new file named middleware.go which will return a `gin.HandleFunc`
```go
const (
    authorizationHeaderKey  = "authorization"
    authorizationTypeBearer = "bearer"
    authorizationPayloadKey = "authorization_payload"
)

func authMiddleware(tokenMaker token.Maker) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader(authorizationHeaderKey)
        if authHeader == "" {
            err := errors.New("authorization header not provided")
            c.AbortWithStatusJSON(401, errorResponse(err))
            return
        }

        authHeaderParts := strings.Fields(authHeader)
        if len(authHeaderParts) < 2 {
            err := errors.New("invalid authorization header")
            c.AbortWithStatusJSON(401, errorResponse(err))
            return
        }

        authorizationType := strings.ToLower(authHeaderParts[0])
        if authorizationType != authorizationTypeBearer {
            err := fmt.Errorf("unsuported auth type %s", authorizationType)
            c.AbortWithStatusJSON(http.StatusUnauthorized, errorResponse(err))
            return
        }

        accessToken := authHeaderParts[1]
        claims, err := tokenMaker.VerifyToken(accessToken)
        if err != nil {
            c.AbortWithStatusJSON(401, errorResponse(err))
            return
        }

		c.Set(authorizationPayloadKey, claims)
        c.Next()
    }
}
```

Now in `server.go` , we will make a new group for all apis which will use this middleware
```go
r := gin.Default()

authRoute := r.Group("/").Use(authMiddleware(server.tokenMaker))

authRoute.PUT("/passengers", server.updatePassenger)
authRoute.GET("/passengers", server.getPassenger)
```

Now, we can use the middleware claims body to get our values
```go
authPayload := c.MustGet(authorizationPayloadKey).(*token.Payload)
// we can use
authPayload.Email
authPayload.Phone
```